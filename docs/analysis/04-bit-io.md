# Bit I/O

## Source Files

- `lib/jxl/padded_bytes.h` (205 lines) -- `PaddedBytes` container: `std::vector<uint8_t>` replacement with +8 bytes padding to allow unaligned 64-bit stores without bounds checks. Cache-line aligned via `AlignedMemory`.
- `lib/jxl/base/byte_order.h` (274 lines) -- Endianness detection and Load/Store helpers (`LoadLE64`, `StoreLE64`, etc.) for little-endian and big-endian memory access. Uses `memcpy` idiom for strict-aliasing safety.
- `lib/jxl/enc_bit_writer.h` (150 lines) -- `BitWriter` struct: accumulates bits into a `PaddedBytes` buffer using direct unaligned 64-bit stores (no intermediate accumulator register).
- `lib/jxl/enc_bit_writer.cc` (208 lines) -- Implementation of `BitWriter::Write`, `Allotment` lifecycle, `AppendByteAligned`, `AppendUnaligned`.
- `lib/jxl/dec_bit_reader.h` (315 lines) -- `BitReader` class: reads bits from a byte buffer using a 64-bit accumulator (`buf_`) with deferred refill strategy.
- `lib/jxl/dec_bit_reader.cc` (36 lines) -- `BoundsCheckedRefill` cold-path implementation.
- `lib/jxl/base/common.h` (170 lines) -- Shared constants: `kBitsPerByte`, `RoundUpBitsToByteMultiple`, `DivCeil`.

## Key Types

### `BitWriter` (struct, `enc_bit_writer.h:31`)

A write-only bit stream that packs bits directly into a byte buffer with no intermediate accumulator. The entire design relies on 64-bit unaligned stores to memory.

**Fields:**
```cpp
size_t bits_written_;              // total bits written so far
PaddedBytes storage_;              // backing byte buffer (+8 padding for overwrite safety)
Allotment* current_allotment_;     // tracks pre-allocated capacity (nested stack)
```

**No accumulator register.** Unlike `BitReader`, `BitWriter` does not maintain a 64-bit bit buffer in a register. Instead, each `Write()` call:
1. Reads the current partial byte from `storage_` into a local `uint64_t`
2. OR's the new bits (shifted left by the sub-byte offset) into that `uint64_t`
3. Writes all 8 bytes back via `memcpy` (on little-endian) -- overwriting up to 7 bytes past the current position

This means the byte at `storage_[bytes_written]` always contains the valid partial bits in its low bits, and the next byte(s) are always zero (maintained as an invariant).

**`kMaxBitsPerCall = 56`:**
```
// Upper bound on `n_bits` in each call to Write. We shift a 64-bit word by
// 7 bits (max already valid bits in the last byte) and at least 1 bit is
// needed to zero-initialize the bit-stream ahead (i.e. if 7 bits are valid
// and we write 57 bits, then the next write will access a byte that was not
// yet zero-initialized).
static constexpr size_t kMaxBitsPerCall = 56;
```
The math: 7 bits already valid in the current byte + 56 new bits = 63 bits total. This fits in 8 bytes (64 bits). The 64th bit (the MSB of the 8th byte) is guaranteed to be zero, maintaining the zero-initialization invariant for the next write. If 57 bits were allowed, 7 + 57 = 64 bits would fill the entire 64-bit store, leaving the next byte uninitialized.

### `BitWriter::Allotment` (nested class, `enc_bit_writer.h:103`)

Pre-allocates space in the `PaddedBytes` buffer before writing, then reclaims unused space afterward. Forms a stack of nested allotments via `parent_` pointers.

**Fields:**
```cpp
size_t prev_bits_written_;   // snapshot of writer->BitsWritten() at Init
const size_t max_bits_;      // max bits this allotment reserved
size_t histogram_bits_ = 0;  // bits used for histogram (optional)
bool called_ = false;        // true after ReclaimAndCharge, prevents double-free
Allotment* parent_;          // linked list for nested allotments
```

**Lifecycle:**
1. `Allotment(max_bits)` -- stores limit
2. `Init(writer)` -- snapshots `bits_written_`, resizes `storage_` by `DivCeil(max_bits_, 8)` bytes, pushes onto allotment stack
3. Writer performs `Write()` calls
4. `ReclaimAndCharge(writer, layer, aux_out)` -- computes `used_bits`, shrinks `storage_` by `unused_bits / 8` whole bytes, pops from stack, adjusts all parent `prev_bits_written_` to avoid double-counting

The `WithMaxBits` method wraps the allotment lifecycle into a single call:
```cpp
Status BitWriter::WithMaxBits(size_t max_bits, LayerType layer, AuxOut* aux_out,
                              const std::function<Status()>& function,
                              bool finished_histogram) {
  BitWriter::Allotment allotment(max_bits);
  JXL_RETURN_IF_ERROR(allotment.Init(this));
  const Status result = function();
  if (result && finished_histogram) {
    JXL_RETURN_IF_ERROR(allotment.FinishedHistogram(this));
  }
  JXL_RETURN_IF_ERROR(allotment.ReclaimAndCharge(this, layer, aux_out));
  return result;
}
```

### `BitReader` (class, `dec_bit_reader.h:29`)

A read-only bit stream with a 64-bit accumulator buffer and deferred refill strategy.

**Fields:**
```cpp
uint64_t buf_;                           // 64-bit accumulator holding pre-read bits
size_t bits_in_buf_;                     // number of valid bits in buf_ [0, 64)
const uint8_t* JXL_RESTRICT next_byte_;  // pointer to next unread byte in source
const uint8_t* end_minus_8_;             // source_end - 8, for fast refill bounds check
const uint8_t* first_byte_;              // start of source, for position tracking
uint64_t overread_bytes_{0};             // bytes "read" past end (injected as zeros)
bool close_called_{false};               // ensures Close() is called exactly once
uint64_t checked_out_of_bounds_bits_{0}; // high-water mark for user-checked OOB
```

**Invariant after Refill:** `56 <= bits_in_buf_ < 64`. This guarantees that any single `ReadBits(n)` call with `n <= 56` can be satisfied without another refill.

**Close protocol:** `BitReader` destructor asserts `close_called_ || !first_byte_`. The user MUST call `Close()` before destruction. `Close()` checks whether more bits were consumed than available in the stream (factoring in `overread_bytes_`). `BitReaderScopedCloser` provides RAII-based close.

### `PaddedBytes` (class, `padded_bytes.h:29`)

A move-only byte container backed by `AlignedMemory` that provides:

1. **+8 byte padding**: `reserve()` allocates `new_capacity + 8` bytes. This allows `BitWriter::Write()` to unconditionally store 8 bytes at any position within `[0, capacity)` without bounds checking.
2. **Zero-initialization of sentinel byte**: On first allocation, `data[0] = 0`. On resize, `data[size_] = 0`. This maintains the invariant that the first unwritten byte is zero, required by `BitWriter::Write()` which reads-then-ORs.
3. **1.5x growth**: `reserve()` grows to `max(capacity, 3 * capacity_ / 2)`, minimum 64 bytes.
4. **Cache-line alignment**: Via `AlignedMemory::Create()`.

**Fields:**
```cpp
JxlMemoryManager* memory_manager_;
size_t size_;
size_t capacity_;
AlignedMemory data_;
```

**Critical bounds check relaxation:**
```cpp
void BoundsCheck(size_t i) const {
  // <= is safe due to padding and required by BitWriter.
  JXL_DASSERT(i <= size());
}
```
The `<=` (not `<`) is intentional -- `BitWriter` accesses `storage_[bytes_written]` where `bytes_written == size()` is legal due to the +8 padding.

## Key Functions

### `BitWriter::Write(size_t n_bits, uint64_t bits)` (`enc_bit_writer.cc:183`)

The core bit-packing function. Writes bits into bytes in **increasing addresses**, **LSB-first within each byte**.

```cpp
void BitWriter::Write(size_t n_bits, uint64_t bits) {
  JXL_DASSERT((bits >> n_bits) == 0);
  JXL_DASSERT(n_bits <= kMaxBitsPerCall);
  size_t bytes_written = bits_written_ / kBitsPerByte;
  uint8_t* p = &storage_[bytes_written];
  const size_t bits_in_first_byte = bits_written_ % kBitsPerByte;
  bits <<= bits_in_first_byte;
#if JXL_BYTE_ORDER_LITTLE
  uint64_t v = *p;
  // Last (partial) or next byte to write must be zero-initialized!
  JXL_DASSERT(v >> bits_in_first_byte == 0);
  v |= bits;
  memcpy(p, &v, sizeof(v));  // Write bytes: possibly more than n_bits/8
#else
  *p++ |= static_cast<uint8_t>(bits & 0xFF);
  for (size_t bits_left_to_write = n_bits + bits_in_first_byte;
       bits_left_to_write >= 9; bits_left_to_write -= 8) {
    bits >>= 8;
    *p++ = static_cast<uint8_t>(bits & 0xFF);
  }
  *p = 0;
#endif
  bits_written_ += n_bits;
}
```

**Little-endian path (the fast path):**
1. Compute byte offset: `bytes_written = bits_written_ / 8`
2. Get pointer to the current partial byte: `p = &storage_[bytes_written]`
3. Compute sub-byte offset: `bits_in_first_byte = bits_written_ % 8`
4. Shift new bits left by `bits_in_first_byte` to align them past existing valid bits
5. Read the current byte at `*p` into a `uint64_t v` (only the low `bits_in_first_byte` bits are non-zero due to the zero-init invariant)
6. OR: `v |= bits`
7. Store all 8 bytes back: `memcpy(p, &v, 8)` -- this overwrites bytes beyond the current position, but those bytes contained zeros and now contain the upper bits of the value

The `memcpy` store is safe because `PaddedBytes` has +8 bytes of padding beyond capacity.

The written byte at `p[ceil((n_bits + bits_in_first_byte) / 8)]` is guaranteed to be zero because:
- At most 7 + 56 = 63 bits are stored
- 63 bits spans 8 bytes, byte index 7 has its MSB = 0
- So the invariant "first unwritten byte is zero" is maintained

**Big-endian path (fallback):**
Writes byte-by-byte with explicit shifting, then zeros the trailing byte.

### `BitReader::Refill()` (`dec_bit_reader.h:84`)

The fast-path refill, inlined at every call site.

```cpp
JXL_INLINE void Refill() {
  if (JXL_UNLIKELY(next_byte_ > end_minus_8_)) {
    BoundsCheckedRefill();
  } else {
    buf_ |= LoadLE64(next_byte_) << bits_in_buf_;
    next_byte_ += (63 - bits_in_buf_) >> 3;
    bits_in_buf_ |= 56;
    JXL_DASSERT(56 <= bits_in_buf_ && bits_in_buf_ < 64);
  }
}
```

**Step by step (fast path, `next_byte_ <= end_minus_8_`):**
1. Load 8 bytes from `next_byte_` as a little-endian 64-bit integer
2. Shift left by `bits_in_buf_` to place new bits above existing valid bits. Since `bits_in_buf_ < 64`, this shift is well-defined. The new bits from the load occupy positions `[bits_in_buf_, bits_in_buf_ + 64)`. Only bits in positions `[bits_in_buf_, 63]` survive (upper bits are truncated by the 64-bit word size). But since we loaded fresh bytes and only care about the lower portion, this is correct.
3. OR into `buf_` -- existing valid bits in `[0, bits_in_buf_)` are preserved
4. Advance `next_byte_` by `(63 - bits_in_buf_) >> 3` bytes. This is the number of complete bytes consumed. Example: if `bits_in_buf_ = 24`, then `(63 - 24) >> 3 = 4`, so 4 bytes (32 bits) are consumed, bringing `bits_in_buf_` up to at least 56.
5. Set `bits_in_buf_ |= 56`: This sets the upper 3 bits of the low 6 bits, guaranteeing the result is in `[56, 63]`. The low 3 bits are preserved because we consumed a whole number of bytes (multiple of 8 bits).

**Key insight on `bits_in_buf_ |= 56`:** The value 56 in binary is `0b111000`. OR-ing this sets bits 3, 4, 5 (the upper three bits of the 6-bit range [0, 63]) without touching bits 0, 1, 2. Since the refill consumed `floor((63 - bits_in_buf_) / 8) * 8` bits (a multiple of 8), the low 3 bits of `bits_in_buf_` remain unchanged. After OR-ing, `bits_in_buf_` has its three MSBs set, giving a value in [56, 63].

### `BitReader::BoundsCheckedRefill()` (`dec_bit_reader.cc:16`)

Cold-path refill when fewer than 8 bytes remain.

```cpp
void BitReader::BoundsCheckedRefill() {
  const uint8_t* end = end_minus_8_ + 8;
  for (; bits_in_buf_ < 64 - kBitsPerByte; bits_in_buf_ += kBitsPerByte) {
    if (next_byte_ >= end) break;
    buf_ |= static_cast<uint64_t>(*next_byte_++) << bits_in_buf_;
  }
  JXL_DASSERT(bits_in_buf_ < 64);

  // Add extra bytes as 0 at the end of the stream.
  size_t extra_bytes = (63 - bits_in_buf_) / kBitsPerByte;
  overread_bytes_ += extra_bytes;
  bits_in_buf_ += extra_bytes * kBitsPerByte;

  JXL_DASSERT(bits_in_buf_ < 64);
  JXL_DASSERT(bits_in_buf_ >= 56);
}
```

1. Reads one byte at a time, OR-ing into `buf_` at position `bits_in_buf_`, advancing `next_byte_`
2. Stops when either `bits_in_buf_ >= 56` or source is exhausted
3. If source is exhausted before reaching 56 bits: injects zero-valued virtual bytes and tracks them in `overread_bytes_`. This allows reads past EOF to return 0 without immediate failure -- the error is deferred to `Close()`.

### `BitReader::ReadBits(size_t nbits)` (`dec_bit_reader.h:146`)

```cpp
JXL_INLINE uint64_t ReadBits(size_t nbits) {
  JXL_DASSERT(!close_called_);
  Refill();
  const uint64_t bits = PeekBits(nbits);
  Consume(nbits);
  return bits;
}
```

Calls `Refill()` to ensure at least 56 bits available, then peeks and consumes. The `PeekBits` implementation:
```cpp
JXL_INLINE uint64_t PeekBits(size_t nbits) const {
#if defined(__BMI2__) && defined(__x86_64__)
  return _bzhi_u64(buf_, nbits);  // zero bits at and above position nbits
#else
  const uint64_t mask = (1ULL << nbits) - 1;
  return buf_ & mask;
#endif
}
```

The `Consume` implementation:
```cpp
JXL_INLINE void Consume(size_t num_bits) {
  bits_in_buf_ -= num_bits;
  buf_ >>= num_bits;
}
```

The pattern is: Refill fills from the top (OR-ing shifted bytes), Peek reads from the bottom (masking LSBs), Consume shifts down (discarding consumed LSBs). This is a classic "LSB-first shift register" pattern.

### `BitWriter::ZeroPadToByte()` (`enc_bit_writer.h:89`)

```cpp
void ZeroPadToByte() {
  const size_t remainder_bits =
      RoundUpBitsToByteMultiple(bits_written_) - bits_written_;
  if (remainder_bits == 0) return;
  Write(remainder_bits, 0);
  JXL_DASSERT(bits_written_ % kBitsPerByte == 0);
}
```

Writes 1-7 zero bits to reach the next byte boundary. Used before operations that require byte alignment (TOC entries, group boundaries).

### `BitReader::JumpToByteBoundary()` (`dec_bit_reader.h:206`)

```cpp
Status JumpToByteBoundary() {
  const size_t remainder = TotalBitsConsumed() % kBitsPerByte;
  if (remainder == 0) return true;
  if (JXL_UNLIKELY(ReadBits(kBitsPerByte - remainder) != 0)) {
    return JXL_FAILURE("Non-zero padding bits");
  }
  return true;
}
```

Reads and discards 1-7 bits to reach byte alignment. **Validates that padding bits are zero** -- returns failure if any padding bit is non-zero. This is a conformance check.

### `BitWriter::AppendByteAligned(Span)` (`enc_bit_writer.cc:111`)

```cpp
Status BitWriter::AppendByteAligned(const Span<const uint8_t>& span) {
  if (span.empty()) return true;
  JXL_RETURN_IF_ERROR(storage_.resize(storage_.size() + span.size() + 1));
  JXL_ENSURE(BitsWritten() % kBitsPerByte == 0);
  size_t pos = BitsWritten() / kBitsPerByte;
  memcpy(storage_.data() + pos, span.data(), span.size());
  pos += span.size();
  JXL_ENSURE(pos < storage_.size());
  storage_[pos++] = 0;  // for next Write
  bits_written_ += span.size() * kBitsPerByte;
  return true;
}
```

Fast byte-copy path for appending pre-built byte data. Requires the writer to be byte-aligned. Zeros the byte after the appended data to maintain the zero-init invariant.

### `BitWriter::AppendUnaligned(const BitWriter& other)` (`enc_bit_writer.cc:127`)

```cpp
Status BitWriter::AppendUnaligned(const BitWriter& other) {
  return WithMaxBits(other.BitsWritten(), LayerType::Header, nullptr, [&] {
    size_t full_bytes = other.BitsWritten() / kBitsPerByte;
    size_t remaining_bits = other.BitsWritten() % kBitsPerByte;
    for (size_t i = 0; i < full_bytes; ++i) {
      Write(8, other.storage_[i]);
    }
    if (remaining_bits > 0) {
      Write(remaining_bits,
            other.storage_[full_bytes] & ((1u << remaining_bits) - 1));
    }
    return true;
  });
}
```

Slow path: copies another writer's bits when not byte-aligned. Reads byte-by-byte from the source and writes via `Write()`. Masks the final partial byte.

### `BitReader::TotalBitsConsumed()` (`dec_bit_reader.h:201`)

```cpp
size_t TotalBitsConsumed() const {
  const size_t bytes_read = static_cast<size_t>(next_byte_ - first_byte_);
  return (bytes_read + overread_bytes_) * kBitsPerByte - bits_in_buf_;
}
```

Computes the logical read position: total bytes advanced (including overread) converted to bits, minus the bits still buffered.

### `BitReader::SkipBits(size_t skip)` (`dec_bit_reader.h:165`)

Efficient skip for large bit counts. First drains the buffer, then advances `next_byte_` by whole bytes, then refills and consumes the remaining fractional bits. Handles overflow (skipping past end) by clamping `next_byte_` to end.

### Endianness Handling (`byte_order.h`)

All bit I/O assumes **little-endian byte order** for the fast path. The bitstream itself is defined as LSB-first within bytes, little-endian across bytes -- which matches native little-endian memory layout.

**`LoadLE64`** (used by `BitReader::Refill`):
```cpp
// On little-endian:
static JXL_INLINE uint64_t LoadLE64(const uint8_t* p) {
  uint64_t little;
  memcpy(&little, p, 8);  // no-op on LE, compiler optimizes to single load
  return little;
}
```

**`JXL_BYTE_ORDER_LITTLE`** detection:
```cpp
#if (defined(__BYTE_ORDER__) && (__BYTE_ORDER__ == __ORDER_LITTLE_ENDIAN__))
#define JXL_BYTE_ORDER_LITTLE 1
#else
#define JXL_BYTE_ORDER_LITTLE 0
#endif
```

On big-endian systems, `BitWriter::Write` falls back to a byte-at-a-time loop, and `BitReader::Refill` uses the byte-at-a-time `BoundsCheckedRefill`-style approach via the generic `LoadLE64` (which reconstructs the little-endian value byte by byte).

## Constants

| Constant | Value | Location | Purpose |
|---|---|---|---|
| `kMaxBitsPerCall` (writer) | 56 | `enc_bit_writer.h:37` | Max bits per `Write()` call. 7 existing + 56 new = 63 < 64. |
| `kMaxBitsPerCall` (reader) | 56 | `dec_bit_reader.h:31` | Max bits per `ReadBits()` call. Matches writer. |
| `kBitsPerByte` | 8 | `common.h:25` | Used throughout instead of `CHAR_BIT`. |
| PaddedBytes min capacity | 64 | `padded_bytes.h:83` | `reserve()` minimum: `std::max<size_t>(64, new_capacity)` |
| PaddedBytes padding | 8 | `padded_bytes.h:88` | Extra bytes allocated: `new_capacity + 8` |
| PaddedBytes growth factor | 1.5x | `padded_bytes.h:82` | `std::max(capacity, 3 * capacity_ / 2)` |
| Refill target bits | 56 | `dec_bit_reader.h:98` | After refill: `bits_in_buf_ |= 56` guarantees >= 56 |
| BMI2 optimization | `_bzhi_u64` | `dec_bit_reader.h:122` | Faster PeekBits when BMI2 available |

## Algorithm Details

### Bit Packing Order

**LSB-first within bytes, little-endian across bytes.**

From `enc_bit_writer.h:80-81`:
> Writes bits into bytes in increasing addresses, and within a byte least-significant-bit first.

Verified by the test `BitReaderTest::TestOrder`:
```cpp
// Writing 5 ones, then 5 zeros, then 6 ones produces:
// Byte 0: 0x1F = 0b00011111  (5 ones in LSBs, 3 zeros in MSBs)
// Byte 1: 0xFC = 0b11111100  (2 zeros in LSBs from the 5 zeros, 6 ones in MSBs)
EXPECT_EQ(0x1Fu, reader.ReadFixedBits<8>());
EXPECT_EQ(0xFCu, reader.ReadFixedBits<8>());
```

And the multi-byte test:
```cpp
// Writing 16-bit value 0xF83F:
// Read byte 0: 0x3F (low byte, stored first)
// Read byte 1: 0xF8 (high byte, stored second)
writer.Write(16, 0xF83F);
EXPECT_EQ(0x3Fu, reader.ReadFixedBits<8>());  // low byte first
EXPECT_EQ(0xF8u, reader.ReadFixedBits<8>());  // high byte second
```

This is standard little-endian bit numbering: bit 0 is the LSB of byte 0, bit 7 is the MSB of byte 0, bit 8 is the LSB of byte 1, etc.

### How the 64-bit Buffer Works (Writer)

The writer has **no persistent accumulator**. Each `Write()` call:

```
Before Write(n=11, bits=0x7FF):
  storage_: [... AA BB 00 00 00 00 00 00 00 ...]
                  ^-- bytes_written points here (byte containing partial bits)
  bits_written_ = 19  (2 full bytes + 3 bits in partial byte)
  bits_in_first_byte = 3
  partial byte BB = 0bxxx00101  (3 valid bits in LSBs)

Step 1: bits <<= 3  ->  0x7FF << 3 = 0x3FF8
Step 2: v = *p = 0x05 (the partial byte, extended to uint64_t)
Step 3: v |= 0x3FF8  ->  v = 0x3FFD
Step 4: memcpy(p, &v, 8)  ->  writes [0xFD, 0x3F, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00]
Step 5: bits_written_ += 11 = 30

After: storage_[2] = 0xFD, storage_[3] = 0x3F, storage_[4..9] = 0x00
```

The zero-initialization invariant after the write: `storage_[DivCeil(bits_written_, 8)]` is guaranteed zero because the 64-bit store wrote zeros past the valid bits.

### How the 64-bit Buffer Works (Reader)

The reader maintains a classic "bit buffer" / "shift register" accumulator:

```
buf_: [63 .......................... 0]
       ^-- unconsumed bits shifted up   |-- next bits to read are at LSB

After Refill (bits_in_buf_ = 58):
buf_ = 0x03FF_FFFF_FFFF_FFFF  (58 valid bits in positions [0, 57], zeros above)

ReadBits(12):
  PeekBits(12): return buf_ & 0xFFF  (extract 12 LSBs)
  Consume(12): buf_ >>= 12; bits_in_buf_ -= 12
  Now bits_in_buf_ = 46, next read will get what was in bits [12, 57]
```

The refill strategy from Fabian Giesen's "variant 4":
- Load 8 bytes from `next_byte_` as LE64
- Shift left by `bits_in_buf_` (places new data above existing valid bits)
- OR into `buf_` (merges without disturbing existing bits)
- Advance `next_byte_` by the number of complete bytes consumed
- Set `bits_in_buf_ |= 56` to indicate at least 56 bits available

This is a "top fill, bottom consume" pattern: new data enters at high bit positions, old data is consumed from low bit positions.

### How Writer Handles Overflow / Growing the Buffer

The writer uses the `Allotment` system to pre-allocate space:

1. Before any write sequence, the caller must create an `Allotment` (or use `WithMaxBits`) specifying the maximum number of bits to be written.
2. `Allotment::Init()` resizes `storage_` to accommodate `prev_bytes + DivCeil(max_bits, 8)`.
3. All `Write()` calls within the allotment are guaranteed to have enough space (no bounds checks needed).
4. `ReclaimAndCharge()` shrinks `storage_` by the unused whole bytes.

If `Write()` is called without a sufficient allotment, the +8 padding on `PaddedBytes` provides a safety margin, but this is only for small overruns. The debug assert `(bits >> n_bits) == 0` catches invalid input but there's no explicit capacity check in `Write()` itself -- correct usage depends on the allotment system.

`PaddedBytes::reserve()` uses 1.5x growth with minimum 64 bytes:
```cpp
size_t new_capacity = std::max(capacity, 3 * capacity_ / 2);
new_capacity = std::max<size_t>(64, new_capacity);
```

### How Reader Handles End-of-Stream

**Deferred error detection:**

1. When `next_byte_ > end_minus_8_`, `Refill()` calls `BoundsCheckedRefill()`.
2. `BoundsCheckedRefill()` reads available bytes one at a time, then injects virtual zero bytes for any shortfall, tracking them in `overread_bytes_`.
3. `PeekBits()` / `ReadBits()` return zeros for the overread portion -- no immediate error.
4. `Close()` checks `TotalBitsConsumed() > TotalBytes() * kBitsPerByte` and returns failure if bits were consumed beyond the actual stream length.
5. `AllReadsWithinBounds()` allows the caller to check and acknowledge the overread condition before `Close()`, preventing a spurious error when the caller handles the condition at a higher level.

The `overread_bytes_` field is also used in `SkipBits()` when skipping past end:
```cpp
if (whole_bytes > static_cast<size_t>(end_minus_8_ + 8 - next_byte_)) {
  next_byte_ = end_minus_8_ + 8;  // clamp to end
  skip += kBitsPerByte;           // account for partial byte
}
```

### Role of `end_minus_8_`

`end_minus_8_ = bytes.data() - 8 + bytes.size()`. This allows the fast refill path to check `next_byte_ <= end_minus_8_` instead of `next_byte_ + 8 <= end`, saving an addition on every refill. When there are at least 8 bytes remaining, the 8-byte `LoadLE64` is safe. When fewer than 8 bytes remain, the bounds-checked slow path reads byte by byte.

## Dependencies

### BitWriter depends on:
- `PaddedBytes` (`padded_bytes.h`) -- backing storage
- `AlignedMemory` (`memory_manager_internal.h`) -- underlying allocation
- `Span` / `Bytes` (`base/span.h`) -- for `GetSpan()` return type
- `byte_order.h` -- `LoadLE64` used in big-endian fallback path (via `#include` in .cc)
- `common.h` -- `kBitsPerByte`, `DivCeil`, `RoundUpBitsToByteMultiple`
- `JxlMemoryManager` (`<jxl/memory_manager.h>`) -- allocation API
- `AuxOut` / `LayerType` (`enc_aux_out.h`) -- bit accounting per layer

### BitReader depends on:
- `byte_order.h` -- `LoadLE64` for fast refill, `LoadLE16` for `BoundsCheckedReadByteAlignedWord`
- `common.h` -- `kBitsPerByte`
- `compiler_specific.h` -- `JXL_INLINE`, `JXL_NOINLINE`, `JXL_RESTRICT`
- `status.h` -- `Status`, `JXL_FAILURE`, `JXL_DASSERT`
- `<immintrin.h>` -- conditional, for `_bzhi_u64` when `__BMI2__` is defined
- No dependency on `PaddedBytes` -- accepts any `ArrayLike` with `.data()` / `.size()`

### PaddedBytes depends on:
- `AlignedMemory` (`memory_manager_internal.h`) -- actual memory allocation
- `JxlMemoryManager` -- pluggable allocator
- `status.h` -- `Status`, `StatusOr`

### Relationship between BitWriter and BitReader:
- Writer uses `kMaxBitsPerCall = 56`, reader also uses `kMaxBitsPerCall = 56`
- Both operate LSB-first within bytes, little-endian across bytes
- The test `BitWriterTest::RandomSequence` writes 1M random patches with `BitWriter`, reads them back with `BitReader`, verifying round-trip correctness
- `JumpToByteBoundary()` (reader) validates that `ZeroPadToByte()` (writer) wrote zeros

## Open Questions

1. **Big-endian support completeness**: The big-endian `Write()` path exists but is it tested? The comment style and `#if` structure suggest it may be legacy/theoretical. All tests appear to run on little-endian only. Is big-endian JPEG XL decoding actually supported in practice?

2. **`BoundsCheckedReadByteAlignedWord()`**: This private method in `BitReader` (`dec_bit_reader.h:256`) reads a 16-bit LE word but is never called from any visible code in these files. Where is it used? It bypasses the normal Refill/Peek/Consume path, suggesting it's for a performance-critical aligned-read shortcut elsewhere (possibly ANS decoding).

3. **Allotment nesting correctness**: The parent adjustment loop in `PrivateReclaim` adjusts `prev_bits_written_` for all ancestors. If an inner allotment writes fewer bits than expected, the parent's accounting is updated. But what happens if the inner allotment writes zero bits? The parent still gets adjusted by zero, which is correct but adds unnecessary traversal.

4. **Maximum stream size**: `bits_written_` is `size_t`, so on 64-bit systems the theoretical max is ~2 exabytes. In practice, `PaddedBytes` is limited by available memory via `JxlMemoryManager`. Is there any explicit size limit enforced?

5. **Thread safety**: Neither `BitWriter` nor `BitReader` has any synchronization. The encoder creates separate `BitWriter` instances per group (via `std::vector<std::unique_ptr<BitWriter>>` in `AppendByteAligned`), then concatenates them. The concatenation presumably happens single-threaded. Is this pattern documented anywhere?

6. **Why 56 and not 57?**: The comment says "at least 1 bit is needed to zero-initialize the bit-stream ahead." But with 7 existing bits + 57 new bits = 64, the memcpy would write exactly 8 bytes with the MSB of byte 7 potentially being a data bit. The NEXT byte (byte 8) would not be zeroed. Since PaddedBytes has +8 padding, this byte exists but would be stale. The 56-bit limit ensures the 8-byte store always includes at least one zero bit at the top, which propagates to the next byte position. This is the conservative choice -- 57 would require zeroing the next byte separately.

7. **`_bzhi_u64` vs mask**: On BMI2-capable CPUs, `PeekBits` uses `_bzhi_u64(buf_, nbits)` instead of `buf_ & ((1ULL << nbits) - 1)`. The `_bzhi` instruction is a single-cycle operation that zeros bits above a specified position. Is the performance difference measurable? The comment says "slightly faster" and notes it's only enabled when the entire binary targets BMI2.

8. **`BitReaderScopedCloser` design**: It takes a reference to `Status` and silently modifies it in the destructor. This is a somewhat unusual RAII pattern. If multiple bit readers share the same status variable, the last close error wins. Is this intentional?

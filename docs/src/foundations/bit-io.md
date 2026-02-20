# Bit I/O

libjxl's bit-level I/O is built for throughput on little-endian hardware. The writer
packs bits directly into memory with 64-bit unaligned stores (no accumulator register).
The reader uses a 64-bit shift register with deferred refill, keeping at least 56 bits
available at all times.

Both operate **LSB-first within bytes, little-endian across bytes** — matching native
x86 memory layout for zero-cost loads and stores.

## Source Files

| File | Lines | Purpose |
|------|-------|---------|
| `enc_bit_writer.h` | 150 | `BitWriter` struct, `Allotment` nested class |
| `enc_bit_writer.cc` | 208 | `Write()` implementation, `Allotment` lifecycle |
| `dec_bit_reader.h` | 315 | `BitReader` class with 64-bit accumulator |
| `dec_bit_reader.cc` | 36 | `BoundsCheckedRefill()` cold path |
| `padded_bytes.h` | 205 | `PaddedBytes` — growable buffer with +8 padding |
| `base/byte_order.h` | 274 | `LoadLE64`, `StoreLE64`, endianness detection |

## Bit Order

```
Byte 0:  [bit7 bit6 bit5 bit4 bit3 bit2 bit1 bit0]
                                              ^^^^
                                         first bits written/read

Byte 1:  [bit15 bit14 bit13 bit12 bit11 bit10 bit9 bit8]
```

Bit 0 is the LSB of byte 0. Bit 7 is the MSB of byte 0. Bit 8 is the LSB of byte 1.
Multi-byte values are stored little-endian: writing `0xF83F` as 16 bits produces
byte 0 = `0x3F`, byte 1 = `0xF8`.

## BitWriter

### Design: No Accumulator

Unlike most bit writers, `BitWriter` has no persistent accumulator register.
Each `Write()` call reads the current partial byte from memory, ORs in new bits,
and writes all 8 bytes back:

```cpp
void BitWriter::Write(size_t n_bits, uint64_t bits) {
    size_t bytes_written = bits_written_ / 8;
    uint8_t* p = &storage_[bytes_written];
    const size_t bits_in_first_byte = bits_written_ % 8;
    bits <<= bits_in_first_byte;

    // Little-endian fast path:
    uint64_t v = *p;                // read partial byte (zeros above valid bits)
    v |= bits;                      // merge new bits
    memcpy(p, &v, sizeof(v));       // store all 8 bytes
    bits_written_ += n_bits;
}
```

The 8-byte `memcpy` overwrites bytes beyond the current position, but those bytes
were zero and now contain the upper bits of the value. The +8 byte padding on
`PaddedBytes` makes this safe even at the end of the buffer.

### kMaxBitsPerCall = 56

Why 56, not 64? At most 7 bits are already valid in the current byte. With 56 new
bits: 7 + 56 = 63 bits, spanning 8 bytes. The 64th bit (MSB of byte 7) is guaranteed
zero, maintaining the invariant that the first unwritten byte is always zero. With
57 bits, all 64 bits of the store could be data, leaving the next byte uninitialized.

### Allotment System

Space is pre-allocated before writing, then reclaimed after:

```cpp
// Typical usage
BitWriter::Allotment allotment(max_bits);
allotment.Init(&writer);       // resize storage, snapshot bits_written_
// ... Write() calls ...
allotment.ReclaimAndCharge(&writer, layer, aux_out);  // shrink unused
```

Allotments form a nested stack via `parent_` pointers. `ReclaimAndCharge` adjusts
all ancestors' accounting when reclaiming unused bytes.

The `WithMaxBits` convenience method wraps the entire lifecycle:

```cpp
writer.WithMaxBits(total_bits, layer, aux_out, [&] {
    // writes go here
    return true;
});
```

## BitReader

### Design: 64-bit Shift Register

The reader maintains a classic "LSB-first shift register":

```
buf_: [63 .......................... 0]
       ^-- fresh bits shifted in      ^-- next bits to read (LSBs)
```

New data enters from the top (via OR with shifted loads). Consumed data exits
from the bottom (via right-shift).

### Refill Strategy

After `Refill()`, at least 56 bits are always available (`bits_in_buf_ ∈ [56, 63]`):

```cpp
void Refill() {
    if (next_byte_ > end_minus_8_) {
        BoundsCheckedRefill();      // cold path: < 8 bytes remain
    } else {
        buf_ |= LoadLE64(next_byte_) << bits_in_buf_;
        next_byte_ += (63 - bits_in_buf_) >> 3;
        bits_in_buf_ |= 56;        // guarantee: [56, 63]
    }
}
```

The `bits_in_buf_ |= 56` trick: 56 = `0b111000`. OR-ing sets bits 3–5 without
touching bits 0–2. Since the refill consumed a whole number of bytes (the low 3 bits
of `bits_in_buf_` are unchanged), the result is in [56, 63].

`end_minus_8_` = `source_end - 8`, saving an addition on every refill check.

### ReadBits

```cpp
uint64_t ReadBits(size_t nbits) {
    Refill();
    uint64_t bits = PeekBits(nbits);  // mask LSBs
    Consume(nbits);                    // buf_ >>= nbits
    return bits;
}
```

`PeekBits` uses `_bzhi_u64(buf_, nbits)` on BMI2-capable CPUs (single-cycle
instruction), falling back to `buf_ & ((1ULL << nbits) - 1)`.

### End-of-Stream Handling

Errors are **deferred**, not immediate:

1. When `< 8` bytes remain, `BoundsCheckedRefill()` reads byte-by-byte
2. If source is exhausted, injects virtual zero bytes, tracking count in `overread_bytes_`
3. `ReadBits()` returns zeros for overread portions — no immediate error
4. `Close()` checks `TotalBitsConsumed() > TotalBytes() * 8` and returns failure

The caller MUST call `Close()` before destruction (enforced by destructor assert).
`BitReaderScopedCloser` provides RAII-based close.

### Position Tracking

```cpp
size_t TotalBitsConsumed() const {
    size_t bytes_read = next_byte_ - first_byte_;
    return (bytes_read + overread_bytes_) * 8 - bits_in_buf_;
}
```

## PaddedBytes

The growable buffer backing `BitWriter`:

| Property | Value |
|----------|-------|
| Extra padding | +8 bytes beyond capacity |
| Minimum capacity | 64 bytes |
| Growth factor | 1.5× |
| Alignment | Via `AlignedMemory` (128+ bytes) |
| Zero init | First unwritten byte always zero |

The `<=` bounds check (instead of `<`) is intentional — `BitWriter` accesses
`storage_[bytes_written]` where `bytes_written == size()` is legal due to padding.

## Byte Alignment

**Writer**: `ZeroPadToByte()` writes 1–7 zero bits to reach the next byte boundary.

**Reader**: `JumpToByteBoundary()` reads 1–7 bits and **validates they are zero** —
a conformance check. Non-zero padding bits are a bitstream error.

## Bulk Operations

`AppendByteAligned(Span)` — fast `memcpy` path for appending pre-built byte data.
Requires the writer to be byte-aligned. Zeros the byte after appended data.

`AppendUnaligned(const BitWriter&)` — slow byte-at-a-time copy from another writer
when not byte-aligned. Used for concatenating group bitstreams.

## Writer/Reader Symmetry

| Property | BitWriter | BitReader |
|----------|-----------|-----------|
| `kMaxBitsPerCall` | 56 | 56 |
| Bit order | LSB-first, little-endian | LSB-first, little-endian |
| Buffer strategy | No accumulator, direct store | 64-bit shift register |
| Alignment | `ZeroPadToByte()` writes zeros | `JumpToByteBoundary()` validates zeros |
| Capacity management | Allotment pre-allocation | Deferred overread detection |
| Error reporting | Via `Status` returns | Deferred to `Close()` |

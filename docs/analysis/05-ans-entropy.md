# ANS Entropy Coding

This document covers the complete ANS (Asymmetric Numeral System) entropy coding
pipeline in libjxl: histogram construction, normalization, alias table building,
symbol encoding/decoding, cost estimation, and the hybrid integer encoding that
wraps the raw ANS/Huffman symbol coding layer.

## Source Files

| File | Lines | Role |
|------|-------|------|
| `lib/jxl/ans_params.h` | 33 | Global constants (table size, alphabet limits, signature) |
| `lib/jxl/ans_common.h` | 157 | Alias table struct, `InitAliasTable` declaration, `GetPopulationCountPrecision`, `CreateFlatHistogram` |
| `lib/jxl/ans_common.cc` | 148 | `InitAliasTable` implementation (Robin Hood alias method) |
| `lib/jxl/dec_ans.h` | 495 | `ANSSymbolReader`, `HybridUintConfig`, `LZ77Params`, `ANSCode`, decoding API |
| `lib/jxl/dec_ans.cc` | 421 | Histogram reading, `DecodeANSCodes`, `DecodeHistograms`, `ANSSymbolReader` construction |
| `lib/jxl/enc_ans.h` | 161 | `ANSEncSymbolInfo`, `ANSCoder`, `Token`, `EntropyEncodingData`, encoding API |
| `lib/jxl/enc_ans.cc` | 1388 | `ANSEncodingHistogram`, histogram normalization/rebalancing, token writing, cost estimation |
| `lib/jxl/enc_ans_params.h` | 185 | `HistogramParams`, `Histogram` struct (encoder-side) |
| `lib/jxl/enc_ans_simd.h` | 24 | `EstimateTokenCost` declaration (SIMD-accelerated) |

## Constants

All defined in `lib/jxl/ans_params.h`:

| Constant | Value | Meaning |
|----------|-------|---------|
| `ANS_LOG_TAB_SIZE` | 12 | Log2 of the ANS probability table size |
| `ANS_TAB_SIZE` | 4096 (1 << 12) | Total probability mass; all symbol frequencies must sum to this |
| `ANS_TAB_MASK` | 4095 | Bitmask for modular indexing into the probability table |
| `ANS_MAX_ALPHABET_SIZE` | 256 | Maximum number of symbols for ANS coding |
| `PREFIX_MAX_ALPHABET_SIZE` | 4096 | Maximum number of symbols for prefix (Huffman) coding |
| `PREFIX_MAX_BITS` | 15 | Maximum Huffman code length in bits |
| `ANS_SIGNATURE` | 0x13 | Initial ANS state value; used as a CRC-like check |

Additional constants from other files:

| Constant | Value | Location | Meaning |
|----------|-------|----------|---------|
| `kClustersLimit` | 128 | `enc_context_map.h` | Maximum number of clustered histograms |
| `kWindowSize` | 1 << 20 (1,048,576) | `dec_ans.h` | LZ77 sliding window size |
| `kNumSpecialDistances` | 120 | `dec_ans.h` | Number of WebP-style special distance codes |
| `RECIPROCAL_PRECISION` | 32 + 12 = 44 | `enc_ans.h` | Bit precision for multiply-by-reciprocal division |
| `kMaxNumSymbolsForSmallCode` | 2 | `enc_ans.cc` | Threshold for "small tree" histogram encoding |
| `Histogram::kRounding` | 8 | `enc_ans_params.h` | SIMD alignment for histogram count vectors |

## Key Types

### `HybridUintConfig` (dec_ans.h:68-103)

The hybrid unsigned integer encoding splits values into a token (entropy-coded)
and extra raw bits (written directly to the bitstream). This is the fundamental
mechanism for encoding arbitrary-size integers through a fixed-alphabet entropy
coder.

```
Fields:
  split_exponent   : uint32  -- values below 2^split_exponent are encoded directly as tokens
  split_token      : uint32  -- = 1 << split_exponent (precomputed)
  msb_in_token     : uint32  -- number of MSBs of the mantissa stored in the token
  lsb_in_token     : uint32  -- number of LSBs of the mantissa stored in the token
```

**Encoding algorithm** (value -> token + extra bits):

For `value < split_token`: `token = value`, `nbits = 0`, `bits = 0`.

For `value >= split_token`:
1. `n = floor(log2(value))` -- the exponent
2. `m = value - 2^n` -- the mantissa (n bits)
3. `token = split_token + ((n - split_exponent) << (msb_in_token + lsb_in_token)) + (m >> (n - msb_in_token)) << lsb_in_token) + (m & ((1 << lsb_in_token) - 1))`
4. `nbits = n - msb_in_token - lsb_in_token`
5. `bits = (value >> lsb_in_token) & ((1 << nbits) - 1)`

The token encodes: the exponent bucket `(n - split_exponent)`, the top
`msb_in_token` bits of the mantissa, and the bottom `lsb_in_token` bits of the
mantissa. The remaining middle bits of the mantissa are stored as raw extra bits.

**Constraint:** `split_exponent >= msb_in_token + lsb_in_token`.

**Default configuration:** `HybridUintConfig(4, 2, 0)` -- direct coding for
0..15, then exponent + 2 MSB mantissa bits in token.

**Decoding** (token + extra bits -> value), from `ReadHybridUintConfig`:

For `token < split_token`: return `token`.

For `token >= split_token`:
1. `nbits = split_exponent - (msb + lsb) + ((token - split_token) >> (msb + lsb))`
2. `low = token & ((1 << lsb) - 1)`
3. `token >>= lsb`
4. Read `nbits` raw bits from the stream
5. `result = (((1 << msb) | (token & ((1 << msb) - 1))) << nbits | bits) << lsb) | low`

### `AliasTable::Entry` (ans_common.h:82-91)

A packed 8-byte struct for constant-time ANS decoding via alias method:

```cpp
#pragma pack(push, 1)
struct Entry {
  uint8_t cutoff;              // boundary within the entry's range
  uint8_t right_value;         // symbol for positions >= cutoff
  uint16_t freq0;              // frequency of the "left" symbol (index i)
  uint16_t offsets1;           // offset for the right_value symbol
  uint16_t freq1_xor_freq0;   // freq1 ^ freq0 (for branchless ternary)
};
#pragma pack(pop)
```

The `freq1_xor_freq0` field enables branchless frequency selection:
`freq = freq0 ^ (greater ? freq1_xor_freq0 : 0)` which equals `freq1` when
`greater` is true and `freq0` when false.

### `AliasTable::Symbol` (ans_common.h:73-77)

Lookup result:

```cpp
struct Symbol {
  size_t value;    // the decoded symbol
  size_t offset;   // cumulative offset within this symbol's range
  size_t freq;     // total frequency of this symbol
};
```

### `ANSEncSymbolInfo` (enc_ans.h:37-47)

Encoder-side per-symbol information:

```cpp
struct ANSEncSymbolInfo {
  uint16_t freq_;                    // ANS frequency (count in [0, ANS_TAB_SIZE])
  std::vector<uint16_t> reverse_map_; // maps cumulative offset -> ANS table position
  uint64_t ifreq_;                   // ceil((1 << 44) / freq_) -- reciprocal for fast division
  uint8_t depth;                     // Huffman code length (prefix coding only)
  uint16_t bits;                     // Huffman code bits (prefix coding only)
};
```

The `reverse_map_` has size equal to `freq_`. For each offset `j` in
`[0, freq_)`, `reverse_map_[j]` gives the ANS table position corresponding to
that offset. Built by iterating over the alias table and recording where each
symbol appears.

### `ANSCoder` (enc_ans.h:49-77)

ANS encoder state machine. Initial state = `ANS_SIGNATURE << 16` = `0x130000`.

```cpp
uint32_t PutSymbol(const ANSEncSymbolInfo& t, uint8_t* nbits) {
    // If state would overflow after encoding, emit 16 low bits first
    if ((state_ >> (32 - ANS_LOG_TAB_SIZE)) >= t.freq_) {
        bits = state_ & 0xffff;
        state_ >>= 16;
        *nbits = 16;
    }
    // Encode: state = (state / freq) * TAB_SIZE + reverse_map[state % freq]
    // Using multiply-by-reciprocal for division:
    const uint32_t v = (state_ * t.ifreq_) >> RECIPROCAL_PRECISION;
    const uint32_t offset = t.reverse_map_[state_ - v * t.freq_];
    state_ = (v << ANS_LOG_TAB_SIZE) + offset;
    return bits;
}
```

### `ANSSymbolReader` (dec_ans.h:162-481)

ANS decoder with LZ77 support. Key state:

```
state_           : uint32  -- ANS decoder state (initialized from 32-bit read for ANS, or ANS_SIGNATURE<<16 for Huffman)
log_alpha_size_  : uint32  -- log2(number of alias table entries per histogram)
log_entry_size_  : uint32  -- = ANS_LOG_TAB_SIZE - log_alpha_size_
entry_size_minus_1_ : uint32 -- = (1 << log_entry_size_) - 1
```

LZ77 state:

```
lz77_window_     : uint32* -- circular buffer of size kWindowSize
num_decoded_     : uint32  -- total symbols decoded so far
num_to_copy_     : uint32  -- remaining LZ77 copy length
copy_pos_        : uint32  -- current position in LZ77 window to copy from
lz77_threshold_  : uint32  -- tokens >= this trigger LZ77 length decode
```

### `ANSCode` (dec_ans.h:147-160)

Decoded entropy code tables shared across all contexts:

```cpp
struct ANSCode {
  AlignedMemory alias_tables;           // packed AliasTable::Entry arrays
  std::vector<HuffmanDecodingData> huffman_data;  // for prefix coding
  std::vector<HybridUintConfig> uint_config;      // per-histogram uint config
  std::vector<int> degenerate_symbols;  // -1 or the single symbol per histogram
  bool use_prefix_code;                 // true = Huffman, false = ANS
  uint8_t log_alpha_size;               // 5..8 for ANS, PREFIX_MAX_BITS for Huffman
  LZ77Params lz77;
  size_t max_num_bits;                  // max bits any ReadHybridUint could return
};
```

### `Token` (enc_ans.h:82-89)

```cpp
struct Token {
  uint32_t is_lz77_length : 1;  // 1 if this is an LZ77 length token
  uint32_t context : 31;        // context index
  uint32_t value;                // the integer value to encode
};
```

### `EntropyEncodingData` (enc_ans.h:96-121)

Encoder-side counterpart of `ANSCode`:

```cpp
struct EntropyEncodingData {
  std::vector<std::vector<ANSEncSymbolInfo>> encoding_info;  // [histogram][symbol]
  bool use_prefix_code;
  std::vector<HybridUintConfig> uint_config;     // per-histogram
  size_t log_alpha_size;
  LZ77Params lz77;
  std::vector<BitWriter> encoded_histograms;     // for streaming mode
  std::vector<uint8_t> context_map;              // maps context -> clustered histogram
};
```

### `Histogram` (enc_ans_params.h:105-181)

Encoder-side frequency histogram:

```cpp
struct Histogram {
  std::vector<ANSHistBin> counts;   // per-symbol counts (ANSHistBin = int32_t)
  size_t total_count = 0;
  mutable float entropy = 0;        // cached, not always up-to-date
  static constexpr size_t kRounding = 8;  // SIMD alignment
};
```

### `HistogramParams` (enc_ans_params.h:30-103)

Controls encoder quality/speed tradeoffs:

```
ClusteringType    : kFastest | kFast | kBest
HybridUintMethod  : kNone | k000 | kFast | kContextMap | kBest
LZ77Method         : kNone | kRLE | kLZ77 | kOptimal
ANSHistogramStrategy : kFast | kApproximate | kPrecise
```

### `LZ77Params` (dec_ans.h:105-120)

```cpp
struct LZ77Params : public Fields {
  bool enabled;
  uint32_t min_symbol;    // symbols >= this are LZ77 lengths; serialized as U32(224|512|4096|BitsOffset(15,8))
  uint32_t min_length;    // minimum match length; serialized as U32(3|4|BitsOffset(2,5)|BitsOffset(8,9))
  HybridUintConfig length_uint_config;  // not serialized by VisitFields, read separately
  size_t nonserialized_distance_context; // last entry in context map
};
```

### `ANSEncodingHistogram` (enc_ans.cc:79-568)

Internal encoder class that finds the best normalized histogram for a given
distribution. Contains the entire logic for histogram normalization,
rebalancing, and bitstream encoding. This is the most complex type in the ANS
system.

Key fields:
```
cost_           : float   -- total estimated cost (header + data bits)
method_         : uint32  -- encoding method (0=flat, 1..ANS_LOG_TAB_SIZE = shift+1 for general)
omit_pos_       : size_t  -- position of the "balancing" symbol (omitted in serialization)
alphabet_size_  : size_t  -- number of symbols with nonzero probability
num_symbols_    : size_t  -- count of distinct nonzero symbols
symbols_[2]     : size_t  -- first two nonzero symbol indices (for small code)
counts_         : vector<ANSHistBin> -- normalized frequencies summing to ANS_TAB_SIZE
```

## Constants and Lookup Tables

### Fixed-Point Log2 LUT (`ANSEncodingHistogram::lg2`)

A table of 4097 entries mapping `i` -> `round(log2(i) / ANS_LOG_TAB_SIZE * 2^31)`.

```cpp
lg2[0] = 0;  // defined as 0 for entropy calculations
lg2[i] = round(ldexp(log2(i) / ANS_LOG_TAB_SIZE, 31));  // for i in [1, 4096]
```

This is fixed-point with 31 fractional bits, normalized by `ANS_LOG_TAB_SIZE`
(12). Used for fast integer entropy estimation.

### Allowed Counts Table (`ANSEncodingHistogram::allowed_counts`)

For each `shift` value in `[0, ANS_LOG_TAB_SIZE)`, precomputes the set of
representable frequency counts and their entropy deltas. Each entry contains:

```cpp
struct CountsEntropy {
  ANSHistBin count : 16;     // the representable count value
  ANSHistBin step_log : 16;  // log2 of step size to next representable value
  int32_t delta_lg2;         // fixed-point change in log2 from this to the next smaller value
};
```

The table is sorted by decreasing count value. The index table `ai[count]`
maps a count value back to its position in the sorted array.

### Histogram Encoding Huffman Table

A hardcoded 128-entry Huffman decode table in `ReadHistogram` (dec_ans.cc:108-125)
for decoding the bit-widths of histogram counts. Format: `{length, symbol}`.

The encoding side uses corresponding fixed tables:

```cpp
constexpr uint8_t kBitWidthLengths[ANS_LOG_TAB_SIZE + 2] = {
    5, 4, 4, 4, 4, 4, 3, 3, 3, 3, 3, 6, 7, 7,
};
constexpr uint8_t kBitWidthSymbols[ANS_LOG_TAB_SIZE + 2] = {
    17, 11, 15, 3, 9, 7, 4, 2, 5, 6, 0, 33, 1, 65,
};
```

Symbol 0 through 12 represent `logcount` values -1 through 11 (where -1 means
count=0). The last symbol (index 13, value `ANS_LOG_TAB_SIZE + 1 = 13`) is
the RLE marker. `kMinReps = 5` is the minimum RLE run length.

## Algorithm Details

### Histogram Serialization Format (Bitstream)

The histogram format has three variants, selected by initial flag bits:

**1. Small Code** (first bit = 1):

- 1 bit: `num_symbols - 1` (0 or 1)
- For each symbol: VarLenUint8 encoding of the symbol index
- If `num_symbols == 2`: `ANS_LOG_TAB_SIZE` (12) bits for the first symbol's count

Covers the common cases of single-symbol distributions (count = ANS_TAB_SIZE)
and two-symbol distributions.

**2. Flat Histogram** (first bit = 0, second bit = 1):

- VarLenUint8: `alphabet_size - 1`
- Counts are implicitly `ANS_TAB_SIZE / alphabet_size` with remainder distributed to first symbols

**3. General Histogram** (first two bits = 00):

- Elias-gamma-like coding of `shift` parameter (method - 1)
- VarLenUint8: `alphabet_size - 3`
- For each symbol: Huffman-coded bit-width (`logcount + 1`) using the static table
  - RLE sequences of >= 5 identical values are coded with the special RLE symbol + VarLenUint8 run length
- The symbol with the largest bit-width is the "omit position" -- its count is
  inferred as `ANS_TAB_SIZE - sum(other counts)`
- For each non-omit symbol with bit-width > 1 (when shift != 0):
  - `bitcount = GetPopulationCountPrecision(logcount, shift)` extra precision bits
  - `count = (1 << logcount) + (read_bits << (logcount - bitcount))`

### `GetPopulationCountPrecision(logcount, shift)` (ans_common.h:26-33)

Determines how many extra bits of precision are stored for a count value whose
floor-log2 is `logcount`, given the current `shift` parameter:

```
r = min(logcount, shift - (ANS_LOG_TAB_SIZE - logcount) / 2)
return max(r, 0)
```

Higher `shift` values allow more precision bits per count, at the cost of more
header bits. The formula balances precision against the diminishing returns for
small counts (where `(ANS_LOG_TAB_SIZE - logcount) / 2` grows large).

### VarLenUint8 Encoding (1-11 bits for values 0..255)

```
value = 0: write bit 0
value > 0: write bit 1, then 3 bits for nbits = floor(log2(value)),
           then nbits bits for value - (1 << nbits)
```

### VarLenUint16 Encoding (1-21 bits for values 0..65535)

Same structure but with 4 bits for nbits instead of 3.

### Alias Table Construction (`InitAliasTable`, ans_common.cc:42-146)

The alias table enables O(1) ANS decoding. It maps the range `[0, ANS_TAB_SIZE)`
to symbols, where each symbol appears exactly `freq` times (its normalized count).

**Parameters:**
- `distribution`: vector of symbol frequencies summing to `range = 1 << log_range`
- `log_range`: typically `ANS_LOG_TAB_SIZE` (12)
- `log_alpha_size`: typically 5..8; determines table layout
- Output: array of `1 << log_alpha_size` `AliasTable::Entry` structs

**Algorithm:**

1. Strip trailing zero-frequency symbols. If empty, add a placeholder symbol
   with weight = range.

2. Compute `entry_size = range >> log_alpha_size`. Each entry covers
   `entry_size` positions in the `[0, range)` range.

3. **Single-symbol special case:** If any symbol has frequency = `ANS_TAB_SIZE`,
   fill all entries pointing to that symbol with `cutoff=0`, `freq0=0`,
   `freq1_xor_freq0=ANS_TAB_SIZE`, `offsets1 = entry_size * i`. This ensures
   the ANS state does not change during decoding.

4. **Robin Hood redistribution (general case):**
   - Initialize `cutoffs[i] = distribution[i]` for each entry.
   - Push entries with `cutoffs[i] > entry_size` onto `overfull_posn` stack.
   - Push entries with `cutoffs[i] < entry_size` onto `underfull_posn` stack.
   - Pad unused entries (beyond distribution.size()) with cutoff=0 and push
     onto underfull stack.
   - While overfull stack is non-empty:
     - Pop one overfull entry `o` and one underfull entry `u`.
     - Fill the empty slots of `u` with symbols from `o`:
       `a[u].right_value = o`
       `a[u].offsets1 = cutoffs[o]` (before adjustment)
       `cutoffs[o] -= (entry_size - cutoffs[u])`
     - If `o` is still overfull or underfull, push to appropriate stack.
   - Final pass: for each entry, if `cutoffs[i] == entry_size`, it's a
     single-symbol entry (set `cutoff=0`, `right_value=i`). Otherwise, adjust
     `offsets1 -= cutoffs[i]` and set `cutoff = cutoffs[i]`.
   - Store `freq0`, `freq1_xor_freq0` for each entry.

**Result:** Each entry represents exactly two symbols. For entry `i`, positions
`[0, cutoff)` map to symbol `i`, positions `[cutoff, entry_size)` map to
`right_value`.

### Alias Table Lookup (`AliasTable::Lookup`, ans_common.h:102-142)

```
Given: value in [0, ANS_TAB_SIZE), log_entry_size, entry_size_minus_1

i = value >> log_entry_size          // which entry
pos = value & entry_size_minus_1     // position within entry
greater = (pos >= cutoff)

symbol.value  = greater ? right_value : i
symbol.offset = (greater ? offsets1 : 0) + pos
symbol.freq   = freq0 ^ (greater ? freq1_xor_freq0 : 0)
```

On little-endian, the entire entry is loaded as a single `uint64_t` and fields
are extracted with shifts and masks, enabling the compiler to use CMOV for the
branchless ternary.

### ANS Decoding Loop (`ReadSymbolANSWithoutRefill`, dec_ans.h:170-197)

```
res = state & ANS_TAB_MASK            // bottom 12 bits
symbol = AliasTable::Lookup(table, res, ...)
state = symbol.freq * (state >> ANS_LOG_TAB_SIZE) + symbol.offset

// Normalization (branchless):
new_state = (state << 16) | PeekFixedBits<16>()
normalize = (state < (1 << 16))
state = normalize ? new_state : state
Consume(normalize ? 16 : 0)

// Prefetch next lookup
next_res = state & ANS_TAB_MASK
AliasTable::Prefetch(table, next_res, log_entry_size)
```

The state transition is: `state' = freq * (state / TAB_SIZE) + offset`, which
is the standard rANS decode step. Normalization reads 16 bits when the state
drops below 2^16.

### ANS Encoding Loop (`WriteTokens`, enc_ans.cc:1237-1321)

ANS encoding processes tokens in **reverse order** (last to first), because
ANS is a stack-based code (LIFO). The output bits are accumulated in reverse
and then written forward.

```
for i = end-1 downto 0:
    encode token.value via HybridUintConfig -> (tok, nbits, bits)
    look up ANSEncSymbolInfo for tok
    // Extra bits first (reversed order)
    addbits(bits, nbits)
    // ANS state update
    ans_bits = ans.PutSymbol(info, &ans_nbits)
    addbits(ans_bits, ans_nbits)

// Write initial state (32 bits), then accumulated bits in reverse
writer->Write(32, ans.GetState())
writer->Write(numallbits, allbits)
for i = out.size()-1 downto 0:
    writer->Write(out_nbits[i], out[i])
```

For prefix (Huffman) coding, tokens are written in forward order since Huffman
codes are prefix-free and don't require reversal.

### ANS Encoder State Update (`ANSCoder::PutSymbol`, enc_ans.h:53-71)

```
// Normalize: emit 16 low bits if state would overflow
if (state >> (32 - ANS_LOG_TAB_SIZE)) >= freq:
    emit low 16 bits of state
    state >>= 16

// Encode using multiply-by-reciprocal:
v = (state * ifreq) >> 44          // v = state / freq (approximately)
offset = reverse_map[state - v * freq]  // state % freq -> table position
state = (v << ANS_LOG_TAB_SIZE) + offset
```

The reciprocal `ifreq = ceil(2^44 / freq)` enables replacing integer division
with a 64-bit multiply and shift.

### ANS Final State Check

The decoder verifies `state == (ANS_SIGNATURE << 16)` = `0x00130000` after
decoding all symbols. This serves as an integrity check.

### Building Encoder Info Table (`ANSBuildInfoTable`, enc_ans.cc:329-353)

After constructing the alias table from normalized counts, this function builds
the `reverse_map_` for each symbol by iterating over all 4096 positions in the
alias table:

```
for i in [0, ANS_TAB_SIZE):
    symbol = AliasTable::Lookup(table, i, ...)
    info[symbol.value].reverse_map_[symbol.offset] = i
```

This creates the inverse mapping needed by the encoder: given a symbol and a
position within that symbol's frequency range, what table position corresponds
to it?

## Cost Functions & Decision Trees

### `ANSEncodingHistogram::EstimateDataBits` (enc_ans.cc:362-369)

Estimates the data bits (excluding header) for encoding a histogram's data
given a set of normalized counts:

```
sum = 0
for each symbol i:
    sum += histo.counts[i] * lg2[normalized_counts[i]]

cost = (total_count - ldexpf(sum, -31)) * ANS_LOG_TAB_SIZE
```

Where `lg2[c]` is the fixed-point table `round(log2(c) / ANS_LOG_TAB_SIZE * 2^31)`.

Expanding: each symbol `i` with original frequency `f_i` and normalized count
`c_i` contributes `f_i * (-log2(c_i / ANS_TAB_SIZE))` bits. The formula
computes this as:

```
cost = sum_i(f_i * (log2(ANS_TAB_SIZE) - log2(c_i)))
     = sum_i(f_i * ANS_LOG_TAB_SIZE) - sum_i(f_i * log2(c_i))
     = total_count * ANS_LOG_TAB_SIZE - sum_i(f_i * log2(c_i))
```

Which matches the implementation: `(total_count - sum/2^31) * 12` where
`sum/2^31 = sum_i(f_i * log2(c_i) / 12)`.

### `ANSEncodingHistogram::EstimateDataBitsFlat` (enc_ans.cc:371-375)

For flat (uniform) histograms:

```
flat_bits = lg2[alphabet_size] * ANS_LOG_TAB_SIZE
cost = ldexpf(total_count * flat_bits, -31)
```

Each symbol costs `log2(alphabet_size)` bits when the distribution is uniform.

### `Histogram::ANSPopulationCost` (enc_ans.cc:619-628)

Returns the total estimated cost (header + data) for encoding this histogram.
Uses `ANSHistogramStrategy::kFast` for speed (tries only shifts 0,
ANS_LOG_TAB_SIZE/2, and ANS_LOG_TAB_SIZE).

### `Histogram::ShannonEntropy` (enc_cluster.cc:223-226)

SIMD-accelerated Shannon entropy calculation:

```
entropy = sum_i(-count_i * log2(count_i / total_count))
```

Implemented via the `Entropy(count, inv_total, total)` function which computes
`-count * FastLog2f(count / total)`, returning 0 when `count == total`.

### `ANSEncodingHistogram::ComputeBest` (enc_ans.cc:84-197)

The main decision function that tries multiple histogram encoding strategies
and picks the cheapest:

1. **Always compute flat histogram cost** as baseline:
   - Flat counts via `CreateFlatHistogram`
   - Cost = header bits + `EstimateDataBitsFlat`

2. **Check for single-symbol histogram** (symbol_count == 1):
   - Set count = ANS_TAB_SIZE for the single symbol
   - Cost = header bits only (no data bits)
   - Return immediately

3. **For 2+ symbols, try different `shift` values** via `RebalanceHistogram`:

   | Strategy | Shifts tried |
   |----------|-------------|
   | `kPrecise` | 0, 1, 2, ..., 11 (all 12 values) |
   | `kApproximate` | 0, 2, 4, 6, 8, 10, 12 (every other) |
   | `kFast` | 0, 6, 12 (three values) |

   For each shift, normalize the histogram, compute header + data cost,
   keep the minimum.

4. Return the best result (minimum total cost).

### `ANSEncodingHistogram::RebalanceHistogram` (enc_ans.cc:416-558)

The core histogram normalization algorithm. Given raw symbol frequencies, produces
normalized counts that sum to `ANS_TAB_SIZE` (4096) and are representable at the
given `shift` precision level.

**Overview:** This is a greedy entropy-maximizing algorithm that adjusts counts
step by step, using a "balancing bin" (the highest-frequency symbol) to absorb
the remaining probability mass.

**Steps:**

1. Compute initial rounded counts: `count = round(freq * ANS_TAB_SIZE / total)`
   clamped to `[1, ANS_TAB_SIZE - 1]` for nonzero frequencies, then snapped
   to the nearest representable value at the given shift precision.

2. Identify the balancing bin (highest raw frequency) and remove it from the
   adjustable set. Its count = `ANS_TAB_SIZE - sum(other counts)`.

3. Build a vector `bins` of adjustable entries with their current positions in
   the `allowed_counts` table.

4. Iteratively optimize:
   - Compute penalty terms `balance_inc[log]` and `balance_dec[log]` that
     represent the entropy cost of changing the balancing bin by `2^log`.
   - Find the bin with the best entropy-per-unit increase (grow step) or the
     bin with the best entropy-per-unit decrease (shrink step).
   - Apply the step: grow the winning bin (decreasing the balancing bin) or
     shrink it (increasing the balancing bin).
   - Stop when no step improves entropy.

5. Handle edge case: if the balancing bin would need `bit_width > 12`, swap it
   with the first bin that has count >= 2048.

6. Set `omit_pos_` to the balancing bin position.

**Entropy delta calculation:**

For increasing a bin from count `c` to the next representable value `c'`:
```
delta = freq * (lg2[c'] - lg2[c]) - max_freq * (lg2[balance] - lg2[balance - (c'-c)])
```

The comparison across different step sizes normalizes by dividing by the step
size: `delta >> step_log`.

### Prefix Code (Huffman) vs ANS Decision

In `BuildAndEncodeHistograms` (enc_ans.cc:1169-1184):

```
use_prefix_code = force_huffman
                  || total_tokens < 100
                  || clustering == kFastest
                  || ans_fuzzer_friendly_

if !use_prefix_code:
    if all histograms have ShannonEntropy < 1e-5:
        use_prefix_code = true   // all are degenerate
```

ANS is preferred for larger streams where its slightly better compression
ratio justifies the 32-bit initial state overhead. Huffman is used for small
streams, fast encoding modes, and degenerate distributions.

### `log_alpha_size` Selection

In `ChooseUintConfigs` (enc_ans.cc:716-726):

```
if use_prefix_code: log_alpha_size = PREFIX_MAX_BITS (15)
elif streaming_mode: log_alpha_size = 8
elif lz77_enabled: log_alpha_size = 8
else: log_alpha_size = 7
```

After `ChooseUintConfigs` determines the actual maximum token, `log_alpha_size`
is adjusted upward to fit (minimum 5, maximum 8 for ANS or 15 for prefix).

The bitstream encodes `log_alpha_size - 5` in 2 bits (for ANS mode), giving
range [5, 8].

### HybridUintConfig Selection (`ChooseUintConfigs`, enc_ans.cc:712-911)

For `HybridUintMethod::kBest`, tries 27 different configurations including
various combinations of `split_exponent` (0-12), `msb_in_token` (0-2), and
`lsb_in_token` (0-5).

For `HybridUintMethod::kFast`, tries 4 configurations:
`(4,2,0)`, `(4,1,2)`, `(0,0,0)`, `(2,0,1)`.

For each configuration, the cost is:
```
cost = ANSPopulationCost(histogram_of_tokens)
     + sum_of_extra_bits
     + CeilLog2Nonzero(split_exponent + 1)      // signaling cost
     + CeilLog2Nonzero(split_exponent - msb + 1) // signaling cost
```

The `EstimateTokenCost` SIMD function computes both the token histogram and
the total extra bits in one pass.

## Bitstream Layout

### Entropy Code Header (per entropy-coded section)

```
1. LZ77Params (Bundle)
   - Bool: enabled
   - If enabled: U32(224|512|4096|BitsOffset(15,8)) for min_symbol
                 U32(3|4|BitsOffset(2,5)|BitsOffset(8,9)) for min_length
                 HybridUintConfig for length_uint_config

2. Context Map (if num_contexts > 1)
   - Encoded via EncodeContextMap/DecodeContextMap

3. 1 bit: use_prefix_code
   - If ANS: 2 bits for (log_alpha_size - 5)

4. Per-histogram HybridUintConfig
   - CeilLog2Nonzero(log_alpha_size+1) bits for split_exponent
   - If split_exponent != log_alpha_size:
     CeilLog2Nonzero(split_exponent+1) bits for msb_in_token
     CeilLog2Nonzero(split_exponent-msb+1) bits for lsb_in_token

5. If prefix coding: per-histogram VarLenUint16 alphabet_size

6. Per-histogram distribution:
   - For prefix: Huffman tree
   - For ANS: histogram in one of the three formats described above

7. If ANS: 32 bits initial state

8. Encoded data (tokens + extra bits)
   - ANS: written in reverse, read forward
   - Prefix: written and read forward
```

### ANS Data Stream Order

The ANS-coded bitstream is laid out as:

```
[32-bit initial state] [reverse-order ANS bits + extra bits interleaved]
```

During encoding, each token produces (in reverse order):
1. Extra raw bits (from HybridUintConfig)
2. ANS normalization bits (0 or 16 bits)

These are concatenated in reverse and written to the stream after the 32-bit
initial state.

## LZ77 Integration

When LZ77 is enabled, an extra context is added for distance codes. Tokens
with symbol >= `lz77.min_symbol` are interpreted as LZ77 length codes:

```
length = ReadHybridUint(lz77_length_uint_config, token - min_symbol) + min_length
distance = ReadHybridUint(distance_context, distance_token)
```

Distance codes 0..119 use the special distance table (WebP-style 2D offsets),
while codes >= 120 represent literal distances offset by `1 - kNumSpecialDistances`.

The LZ77 window is a circular buffer of `kWindowSize` (1M) decoded values. A
`Checkpoint` mechanism (dec_ans.h:403-450) allows saving and restoring the
decoder state for backtracking, with a maximum checkpoint interval of 512
symbols.

The `IsSingleValueAndAdvance` optimization (dec_ans.h:381-401) detects when a
context has a degenerate distribution (single symbol, freq = ANS_TAB_SIZE) and
the token is small enough to be directly used. It advances the decoder state
for `count` symbols without actually decoding them.

## Dependencies

### Internal
- `lib/jxl/dec_bit_reader.h` -- `BitReader` for reading from bitstream
- `lib/jxl/enc_bit_writer.h` -- `BitWriter` for writing to bitstream
- `lib/jxl/dec_huffman.h` -- `HuffmanDecodingData` for prefix code decoding
- `lib/jxl/enc_huffman.h` -- `BuildAndStoreHuffmanTree` for prefix code encoding
- `lib/jxl/enc_cluster.h` -- `ClusterHistograms` for context clustering
- `lib/jxl/enc_context_map.h` -- `EncodeContextMap`, `kClustersLimit`
- `lib/jxl/dec_context_map.h` -- `DecodeContextMap`
- `lib/jxl/enc_lz77.h` -- `ApplyLZ77` for LZ77 preprocessing
- `lib/jxl/enc_ans_simd.h` / `.cc` -- SIMD-accelerated `EstimateTokenCost`, `MaxValue`
- `lib/jxl/fields.h` -- `Bundle::Read/Write` for `LZ77Params` serialization
- `lib/jxl/memory_manager_internal.h` -- `AlignedMemory` for aligned allocations

### External
- Highway (`hwy/`) -- SIMD primitives for alias table prefetch, entropy computation

## Open Questions

1. **Alias table construction order:** The comment at `ans_common.h:68-70` notes
   that the order used for computing offsets is defined by the Robin Hood
   algorithm and is not straightforward to change. The implicit assumption is
   that symbols after the cutoff have offsets >= cutoff. Changing this would
   break decoder compatibility.

2. **log_alpha_size = 14 tradeoff:** The comment in `ans_params.h:14-17` notes
   that `ANS_LOG_TAB_SIZE = 14` gives 0.2% improvement at d1 but hurts d8, and
   "requires recomputing the Huffman tables." The hardcoded Huffman table in
   `ReadHistogram` would need to be regenerated for a different table size.

3. **RLE in uint configs:** Both `EncodeUintConfigs` and `DecodeUintConfigs` have
   TODO comments about adding RLE compression for repeated uint configs.

4. **LZ77 uint config handling:** `ChooseUintConfigs` has a TODO noting it does
   not optimize the uint config for LZ77 distance tokens.

5. **Fuzzer-friendly mode:** When enabled, forces `HybridUintConfig(7,0,0)` or
   `(10,0,0)`, `lz77.min_symbol = 2048`, and uniform histograms. This negatively
   impacts compression but makes the stream more amenable to fuzzing.

6. **RebalanceHistogram optimality:** The greedy entropy-maximizing scheme in
   `RebalanceHistogram` is acknowledged to not guarantee global optimality
   (enc_ans.cc:413-415), but it cannot produce invalid histograms.

7. **Streaming mode constraints:** Streaming mode forces `log_alpha_size = 8`
   and uses the default HybridUintConfig without optimization
   (enc_ans.cc:741-743). The comment suggests investigating whether lower
   values are possible.

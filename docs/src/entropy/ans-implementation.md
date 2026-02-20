# ANS Implementation

This chapter covers the encoder and decoder implementation details: state machines,
cost estimation, histogram normalization, and the encoding/decoding loops.

## Encoder State Machine

```cpp
struct ANSCoder {
    uint32_t state_;  // initialized to ANS_SIGNATURE << 16 = 0x130000
};
```

### PutSymbol — Encode One Symbol

```cpp
uint32_t PutSymbol(const ANSEncSymbolInfo& t, uint8_t* nbits) {
    // Normalize: emit 16 low bits if state would overflow
    if ((state_ >> (32 - ANS_LOG_TAB_SIZE)) >= t.freq_) {
        bits = state_ & 0xFFFF;
        state_ >>= 16;
        *nbits = 16;
    }

    // Encode using multiply-by-reciprocal for fast division:
    const uint32_t v = (state_ * t.ifreq_) >> RECIPROCAL_PRECISION;
    const uint32_t offset = t.reverse_map_[state_ - v * t.freq_];
    state_ = (v << ANS_LOG_TAB_SIZE) + offset;
    return bits;
}
```

`ifreq_ = ceil(2^44 / freq_)` enables replacing integer division with a 64-bit
multiply and 44-bit right shift (`RECIPROCAL_PRECISION = 44`).

The `reverse_map_` (size = `freq_`) maps each offset within the symbol's frequency
range to the corresponding alias table position.

### Encoding Loop (WriteTokens)

ANS encoding processes tokens in **reverse order** (LIFO):

```
for i = end-1 downto 0:
    encode value via HybridUintConfig → (token, nbits, bits)
    look up ANSEncSymbolInfo for token
    accumulate extra bits (reversed)
    accumulate ANS bits from PutSymbol (reversed)

write 32-bit initial state
write all accumulated bits in reverse
```

For prefix (Huffman) coding, tokens are written forward — no reversal needed.

## Decoder State Machine

### ReadSymbolANSWithoutRefill

```
res = state & ANS_TAB_MASK           // bottom 12 bits
symbol = AliasTable::Lookup(table, res, ...)
state = symbol.freq * (state >> ANS_LOG_TAB_SIZE) + symbol.offset

// Branchless normalization:
new_state = (state << 16) | PeekFixedBits<16>()
normalize = (state < (1 << 16))
state = normalize ? new_state : state
Consume(normalize ? 16 : 0)

// Prefetch next lookup
Prefetch(table, state & ANS_TAB_MASK)
```

The decoder verifies `state == (ANS_SIGNATURE << 16)` after all symbols are decoded.

### ANSSymbolReader State

```cpp
uint32_t state_;                 // ANS state (init from 32-bit read)
uint32_t log_alpha_size_;        // log2(entries per histogram)
uint32_t log_entry_size_;        // ANS_LOG_TAB_SIZE - log_alpha_size
uint32_t entry_size_minus_1_;    // (1 << log_entry_size) - 1
```

`log_alpha_size` ranges from 5 to 8 for ANS (encoded as 2 bits: `log_alpha_size - 5`),
or 15 for prefix coding.

## Histogram Normalization

### RebalanceHistogram (enc_ans.cc:416-558)

The core algorithm that normalizes raw frequencies to sum to exactly `ANS_TAB_SIZE`
(4096) while maximizing entropy at a given precision level.

**Steps:**

1. **Initial rounding**: `count = round(freq * 4096 / total)`, clamped to [1, 4095]
   for nonzero symbols, snapped to nearest representable value at given `shift`

2. **Balancing bin**: The highest-frequency symbol absorbs the remainder
   (`4096 - sum(others)`)

3. **Greedy optimization**: Iteratively adjust counts to maximize entropy:
   - For each adjustable bin, compute entropy gain/loss of stepping to next
     representable value
   - Apply the best step (considering the balancing bin's entropy change)
   - Stop when no step improves total entropy

4. **Omit position**: The balancing bin is the "omit position" — its count is
   not serialized (inferred from `4096 - sum`)

### Cost Estimation

**Data bits** (`EstimateDataBits`):
```
cost = (total_count - ldexpf(sum, -31)) * ANS_LOG_TAB_SIZE
where sum = Σ counts[i] * lg2[normalized_counts[i]]
      lg2[c] = round(log2(c) / 12 * 2^31)  // fixed-point LUT
```

This computes `Σ f_i * (-log2(c_i / 4096))` — the cross-entropy between the raw
and normalized distributions.

**Flat distribution** (`EstimateDataBitsFlat`):
```
cost = total_count * log2(alphabet_size)
```

### Strategy Selection (ComputeBest)

Tries multiple histogram encoding strategies and picks the cheapest:

| Strategy | Shifts tried |
|----------|-------------|
| `kPrecise` | 0, 1, 2, ..., 11 (all) |
| `kApproximate` | 0, 2, 4, 6, 8, 10, 12 (every other) |
| `kFast` | 0, 6, 12 (three values) |

For each shift: normalize histogram, compute header + data cost, keep minimum.

Always computes flat histogram cost as a baseline. Single-symbol histograms get
special handling (header only, no data bits).

## Encoder-Side Types

### ANSEncSymbolInfo

```cpp
struct ANSEncSymbolInfo {
    uint16_t freq_;          // normalized frequency [0, 4096]
    vector<uint16_t> reverse_map_;  // offset → table position
    uint64_t ifreq_;         // ceil(2^44 / freq) for fast division
    uint8_t depth;           // Huffman code length (prefix only)
    uint16_t bits;           // Huffman code bits (prefix only)
};
```

### EntropyEncodingData

```cpp
struct EntropyEncodingData {
    vector<vector<ANSEncSymbolInfo>> encoding_info;  // [histogram][symbol]
    bool use_prefix_code;
    vector<HybridUintConfig> uint_config;  // per-histogram
    size_t log_alpha_size;
    LZ77Params lz77;
    vector<uint8_t> context_map;  // context → clustered histogram
};
```

### Token

```cpp
struct Token {
    uint32_t is_lz77_length : 1;  // LZ77 length flag
    uint32_t context : 31;        // context index
    uint32_t value;                // integer value to encode
};
```

## Building the Reverse Map

After alias table construction, `ANSBuildInfoTable` builds the encoder's reverse
map by iterating over all 4096 positions:

```
for i in [0, 4096):
    symbol = AliasTable::Lookup(table, i)
    info[symbol.value].reverse_map_[symbol.offset] = i
```

## Bitstream Layout

```
[LZ77Params]                    // Bundle serialization
[Context Map]                   // if num_contexts > 1
[1 bit: use_prefix_code]
[2 bits: log_alpha_size - 5]    // ANS only
[Per-histogram HybridUintConfig]
[Per-histogram alphabet_size]   // prefix only
[Per-histogram distribution]    // small/flat/general format
[32-bit initial state]          // ANS only
[Encoded data]                  // tokens + extra bits
```

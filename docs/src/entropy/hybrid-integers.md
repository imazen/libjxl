# Hybrid Integer Encoding

JPEG XL uses a split-exponent scheme to encode arbitrary-size integers through a
fixed-alphabet entropy coder. Small values are encoded directly as tokens. Larger
values are split into an entropy-coded token (carrying the exponent and some mantissa
bits) plus raw extra bits written directly to the bitstream.

## The Split

```
value < split_token  →  token = value (direct, no extra bits)
value >= split_token →  token encodes exponent + partial mantissa
                        extra bits encode remaining mantissa
```

## HybridUintConfig

```cpp
struct HybridUintConfig {
    uint32_t split_exponent;   // values below 2^split_exponent are direct
    uint32_t split_token;      // = 1 << split_exponent (precomputed)
    uint32_t msb_in_token;     // MSBs of mantissa stored in the token
    uint32_t lsb_in_token;     // LSBs of mantissa stored in the token
};
// Constraint: split_exponent >= msb_in_token + lsb_in_token
```

**Default**: `HybridUintConfig(4, 2, 0)` — direct coding for 0–15, then exponent
+ 2 MSB mantissa bits in token.

## Encoding Algorithm

For `value >= split_token`:

```
n = floor(log2(value))          // the exponent
m = value - 2^n                 // the mantissa (n bits wide)

token = split_token
      + (n - split_exponent) << (msb_in_token + lsb_in_token)
      + (m >> (n - msb_in_token)) << lsb_in_token
      + (m & ((1 << lsb_in_token) - 1))

nbits = n - msb_in_token - lsb_in_token
bits = (value >> lsb_in_token) & ((1 << nbits) - 1)
```

The token packs: the exponent bucket, the top `msb_in_token` mantissa bits, and
the bottom `lsb_in_token` mantissa bits. The remaining middle bits are raw.

## Decoding Algorithm

For `token >= split_token`:

```
nbits = split_exponent - (msb + lsb)
      + ((token - split_token) >> (msb + lsb))
low = token & ((1 << lsb) - 1)
token >>= lsb
Read nbits raw bits from stream
result = (((1 << msb) | (token & ((1 << msb) - 1))) << nbits | bits) << lsb) | low
```

## Configuration Selection

The encoder tries multiple configurations and picks the cheapest (total cost =
entropy-coded tokens + raw extra bits + signaling overhead):

### kBest — 27 configurations
Various combinations of `split_exponent` (0–12), `msb_in_token` (0–2),
`lsb_in_token` (0–5).

### kFast — 4 configurations
`(4,2,0)`, `(4,1,2)`, `(0,0,0)`, `(2,0,1)`

### Cost Formula

```
cost = ANSPopulationCost(token_histogram)
     + total_extra_bits
     + CeilLog2Nonzero(split_exponent + 1)        // signaling
     + CeilLog2Nonzero(split_exponent - msb + 1)  // signaling
```

`EstimateTokenCost` (SIMD-accelerated) computes both the token histogram and
total extra bits in a single pass.

## Tradeoffs

- **More bits in token** → larger alphabet, higher entropy coding overhead, but
  fewer raw bits (which get no compression)
- **Fewer bits in token** → smaller alphabet, lower overhead, but more uncompressed
  raw bits
- **`(0,0,0)`** = every value is its own token (good for small alphabets)
- **`(4,2,0)`** = direct for 0–15, then exponent + 2 MSBs (the default, good
  general-purpose balance)

## VarLenUint8 and VarLenUint16

Simpler fixed configurations used for small values in headers:

### VarLenUint8 (values 0–255, 1–11 bits)
```
0       → 1 bit
1–255   → 1 + 3 + floor(log2(value)) bits
```

### VarLenUint16 (values 0–65535, 1–21 bits)
Same structure with 4 bits for the exponent instead of 3.

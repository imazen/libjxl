# Coefficient Order

The coefficient order system controls the scan order of DCT coefficients within
each AC block. Rather than always using the standard zig-zag order, the encoder
can compute a custom permutation that places coefficients with more zeros later
in the sequence, improving entropy coding density.

## Source Files

| File | Role |
|------|------|
| `lib/jxl/coeff_order_fwd.h` | Forward declarations, `coeff_order_t` typedef, `kNumOrders`, `CoefficientLayout` |
| `lib/jxl/coeff_order.h` | Shared constants (`kCoeffOrderOffset`, `kStrategyOrder`, `kPermutationContexts`), decoder API |
| `lib/jxl/coeff_order.cc` | Decoder: `DecodeCoeffOrders`, `DecodePermutation`, Lehmer code decoding |
| `lib/jxl/enc_coeff_order.h` | Encoder API: `ComputeUsedOrders`, `ComputeCoeffOrder`, `EncodeCoeffOrders` |
| `lib/jxl/enc_coeff_order.cc` | Encoder: zero-counting, sort-based optimization, Lehmer code encoding |
| `lib/jxl/lehmer_code.h` | Lehmer code encode/decode via Fenwick tree (O(N log N)) |
| `lib/jxl/ac_strategy.cc` | `CoeffOrderAndLut` - natural (zig-zag) order generation for all block sizes |

## Key Types

### `coeff_order_t` (coeff_order_fwd.h:20)

```cpp
using coeff_order_t = uint32_t;
```

The element type for coefficient order arrays. 32 bits for speed (2% faster
`DecodeAC` vs 16-bit at the cost of more memory). The Lehmer code type is also
`uint32_t`:

```cpp
using LehmerT = uint32_t;
```

### `PosAndCount` (enc_coeff_order.cc:155-159)

Internal struct used during sort-based order optimization:

```cpp
struct PosAndCount {
    uint32_t pos;
    uint64_t count_and_idx;  // (quantized_zero_count << 16) | original_index
};
```

The `count_and_idx` field packs the quantized zero count into the upper 48 bits
and the original position index into the lower 16 bits. This ensures that
`std::sort` produces a stable ordering: coefficients with the same zero count
preserve their original (natural/zig-zag) relative order.

### `HybridUintConfig(0, 0, 0)` context mapping

Used in `CoeffOrderContext` to map permutation values to entropy contexts.
With parameters `(split_exponent=0, msb_in_token=0, lsb_in_token=0)`, the
config has `split_token = 1`, so value 0 maps to token 0, and all values >= 1
produce tokens based on `floor(log2(value))`. The result is clamped to
`[0, kPermutationContexts - 1]` = `[0, 7]`.

## Key Functions

### `ComputeUsedOrders` (enc_coeff_order.cc:37-64)

Scans the AC strategy image to determine which order buckets are present in
the image and which should receive custom (non-default) orders.

**Speed gating:**
- `speed >= kFalcon` (speed 7+): returns `{1, 1}` -- only DCT8 order, and it
  is eligible for customization. In practice this means the fastest tiers use
  only DCT8 with default order since no optimization runs.

**Size gating:**
- If the AC strategy image is smaller than 5x5 blocks (i.e., images smaller
  than ~40x40 pixels), returns `{used, 0}` -- all orders are marked "used" but
  none are customized. The default natural order is used for all.

**Block size gating:**
- Orders with bucket index > 6 (the large transforms: 64x64, 64x32/32x64,
  128x128, 128x64/64x128, 256x256, 256x128/128x256) are never customized.
  They appear in the "used" mask but not in the "customize" mask.

Returns `pair<uint32_t, uint32_t>`:
- First: bitmask of which order buckets are present in the image
- Second: bitmask of which order buckets should be customized

### `ComputeCoeffOrder` (enc_coeff_order.cc:66-237)

The main encoder optimization function. For each order bucket that is marked
for customization, counts zero coefficients per DCT band position across the
entire image, then sorts bands by zero count to produce the optimal scan order.

**Algorithm:**

1. **Block sampling** (lines 77-100): At `speed >= kSquirrel` with only DCT8
   in use (`current_used_orders == 1`), samples only 50% of blocks using
   Xorshift128+ PRNG. The threshold is `(2^32 - 1) * block_fraction` applied
   to the upper 32 bits of the PRNG output.

2. **Zero counting** (lines 102-153): Iterates all groups and blocks. For each
   sampled block, counts zeros per coefficient position within each channel.
   The count accumulates in `num_zeros[order_offset + k]`. Supports both 16-bit
   and 32-bit AC coefficient storage.

3. **LLF protection** (lines 141-148): The low-frequency (LLF) coefficients
   (the DC-like coefficients in multi-block transforms) are forced to sort first
   by setting `num_zeros = -1` for positions `(iy, ix)` where
   `iy < cy, ix < cx` (the covered blocks dimensions, after layout
   normalization).

4. **Quantized sort** (lines 199-229): For each order bucket and channel:
   - Computes `inv_sqrt_sz = 1.0 / sqrt(block_size)`
   - Quantizes: `count = floor(num_zeros[pos] * inv_sqrt_sz + 0.1)`
   - Packs into `PosAndCount` with `count_and_idx = (count << 16) | index`
   - Sorts by `count_and_idx` (ascending)
   - Coefficients with fewer zeros (more likely nonzero) come first
   - The `+ 0.1` bias prevents floating-point truncation artifacts
   - The `<< 16` shift with `| index` ensures stable sort behavior

5. **Default-order detection** (lines 231-233): If the computed order matches
   the natural order for all three channels, the order bucket is removed from
   `current_used_orders` to avoid encoding a no-op permutation.

### `EncodeCoeffOrders` (enc_coeff_order.cc:293-332)

Serializes all custom coefficient orders into the bitstream.

For each order bucket that is flagged in `used_orders`:
1. Converts from absolute position to natural-order-relative index using
   `ComputeNaturalCoeffOrderLut`
2. Tokenizes via `TokenizePermutation` (Lehmer code)
3. All tokens go into a single token stream

The entire stream is entropy-coded using ANS with `kPermutationContexts = 8`
context channels. If `used_orders == 0`, nothing is written.

### `TokenizePermutation` (enc_coeff_order.cc:241-259)

Converts a permutation to a Lehmer code token stream:

1. Computes Lehmer code via `ComputeLehmerCode` (Fenwick tree, O(N log N))
2. Finds the effective end by trimming trailing zeros
3. Emits `(end - skip)` as the first token (the permutation length)
4. Emits each Lehmer code value, using the previous value as entropy context

The context for each token is `CoeffOrderContext(last_value)`, which maps
through `HybridUintConfig(0,0,0).Encode()` then clamps to `[0, 7]`.

### `DecodeCoeffOrders` (coeff_order.cc:102-156)

Decoder-side mirror. Reads permutations from the bitstream:

1. If `used_orders != 0`, decodes ANS histograms for `kPermutationContexts`
   contexts
2. For each AC strategy (deduplicated via `kStrategyOrder`):
   - If the order is not in `used_orders`: copies the natural order
   - If the order is in `used_orders`: decodes 3 permutations (one per channel),
     then maps through `natural_order[]` to convert from permutation indices to
     coefficient positions

### `CoeffOrderAndLut` (ac_strategy.cc:29-79)

Generates the natural (default) coefficient order for any block size. This is
a generalized zig-zag that works for non-square blocks.

**Algorithm:**
- For a block covering `cx * cy` 8x8 sub-blocks (with `cx >= cy` after layout
  normalization), compute zig-zag for a `cx*8 x cx*8` square
- Discard rows that are not multiples of `xs = cx / cy` (the aspect ratio)
- The first `cx * cy` positions are the LLF coefficients (DC-like)
- Remaining positions follow the zig-zag with alternating diagonal direction

When `is_lut = false`: `out[val] = position` (order: index -> coefficient)
When `is_lut = true`: `out[position] = val` (lut: coefficient -> index)

## Constants

### `kNumOrders = 13` (coeff_order_fwd.h:25)

Maximum number of distinct order buckets. One per "size class" plus an extra
for DCT8. Shared between transforms of size X*Y and Y*X.

### `kCoeffOrderLimit = 6156` (coeff_order.h:26)

Total number of coefficient positions across all 13 order buckets, all 3
channels. This is measured in units of `kDCTBlockSize` (64).

### `kCoeffOrderMaxSize = 6156 * 64 = 393,984` (coeff_order.h:39-40)

Total allocation size for the entire coefficient order array.

### `kCoeffOrderOffset` (coeff_order.h:28-33)

A table of 40 entries (13 orders * 3 channels + 1 sentinel) giving the byte
offset (in units of `kDCTBlockSize = 64`) for each (order, channel) pair:

```
{0, 1, 2, 3, 4, 5, 6, 10, 14, 18, 34, 50, 66, 68, 70, 72, 76, 80, 84,
 92, 100, 108, 172, 236, 300, 332, 364, 396, 652, 908, 1164, 1292, 1420,
 1548, 2572, 3596, 4620, 5132, 5644, 6156}
```

Access macro: `CoeffOrderOffset(O, C) = kCoeffOrderOffset[3*O + C] * 64`

From these offsets, the per-order sizes (in coefficients) can be derived:

| Order | Blocks | Size (coeffs) | Transform types |
|-------|--------|---------------|-----------------|
| 0 | 1x1 | 64 | DCT8 |
| 1 | 1x1 | 64 | IDENTITY, DCT2x2, DCT4x4, DCT4x8, DCT8x4, AFV0-3 |
| 2 | 2x2 | 256 | DCT16x16 |
| 3 | 4x4 | 1024 | DCT32x32 |
| 4 | 1x2 | 128 | DCT16x8, DCT8x16 |
| 5 | 1x4 | 256 | DCT32x8, DCT8x32 |
| 6 | 2x4 | 512 | DCT32x16, DCT16x32 |
| 7 | 8x8 | 4096 | DCT64x64 |
| 8 | 4x8 | 2048 | DCT64x32, DCT32x64 |
| 9 | 16x16 | 16384 | DCT128x128 |
| 10 | 8x16 | 8192 | DCT128x64, DCT64x128 |
| 11 | 32x32 | 65536 | DCT256x256 |
| 12 | 16x32 | 32768 | DCT256x128, DCT128x256 |

Note: Orders 0 and 1 both have the same coefficient count (64) but are separate
buckets because DCT8 has a different natural order than IDENTITY/DCT2x2/etc.
The "Blocks" column shows the normalized layout (rows <= columns) after
`CoefficientLayout`; transpose pairs share a bucket.

### `kStrategyOrder` (coeff_order.h:44-47)

Maps each of the 27 `AcStrategyType` values to an order bucket index:

```cpp
constexpr std::array<uint8_t, 27> kStrategyOrder = {
    0,   // [0]  DCT        (8x8,   1x1 blocks)
    1,   // [1]  IDENTITY   (8x8,   1x1 blocks)
    1,   // [2]  DCT2X2     (8x8,   1x1 blocks)
    1,   // [3]  DCT4X4     (8x8,   1x1 blocks)
    2,   // [4]  DCT16X16   (16x16, 2x2 blocks)
    3,   // [5]  DCT32X32   (32x32, 4x4 blocks)
    4,   // [6]  DCT16X8    (16x8,  1x2 blocks)
    4,   // [7]  DCT8X16    (8x16,  1x2 blocks)
    5,   // [8]  DCT32X8    (32x8,  1x4 blocks)
    5,   // [9]  DCT8X32    (8x32,  1x4 blocks)
    6,   // [10] DCT32X16   (32x16, 2x4 blocks)
    6,   // [11] DCT16X32   (16x32, 2x4 blocks)
    1,   // [12] DCT4X8     (4x8,   1x1 blocks)
    1,   // [13] DCT8X4     (8x4,   1x1 blocks)
    1,   // [14] AFV0       (8x8,   1x1 blocks)
    1,   // [15] AFV1       (8x8,   1x1 blocks)
    1,   // [16] AFV2       (8x8,   1x1 blocks)
    1,   // [17] AFV3       (8x8,   1x1 blocks)
    7,   // [18] DCT64X64   (64x64,   8x8 blocks)
    8,   // [19] DCT64X32   (64x32,   4x8 blocks)
    8,   // [20] DCT32X64   (32x64,   4x8 blocks)
    9,   // [21] DCT128X128 (128x128, 16x16 blocks)
    10,  // [22] DCT128X64  (128x64,  8x16 blocks)
    10,  // [23] DCT64X128  (64x128,  8x16 blocks)
    11,  // [24] DCT256X256 (256x256, 32x32 blocks)
    12,  // [25] DCT256X128 (256x128, 16x32 blocks)
    12,  // [26] DCT128X256 (128x256, 16x32 blocks)
};
```

Transform pairs that differ only in orientation (e.g., DCT16X8 and DCT8X16)
share the same order bucket because `CoefficientLayout` normalizes them to the
same layout (shorter dimension = rows).

### `kPermutationContexts = 8` (coeff_order.h:49)

Number of entropy coding contexts for the Lehmer code stream. Each Lehmer code
value is entropy-coded using a context derived from the previous value via
`CoeffOrderContext`.

### `kBlockDim = 8`, `kDCTBlockSize = 64` (frame_dimensions.h:21-23)

Fundamental block dimensions.

### `AcStrategy::kMaxCoeffBlocks = 32` (ac_strategy.h:84)

Maximum number of 8x8 blocks in one dimension of any AC strategy.

### `AcStrategy::kMaxCoeffArea = 256 * 256 = 65536` (ac_strategy.h:88)

Maximum number of coefficients in any single AC strategy block (256x256 DCT).

## Cost Functions & Decision Trees

### Order Optimization Cost Model

The coefficient order optimization does not use an explicit cost function.
Instead, it uses a heuristic based on the observation that placing
zero-heavy coefficient bands later in the scan order improves entropy coding
because:

1. Run-length coding of zeros becomes more efficient when zeros cluster at the
   end of the scan
2. The end-of-block (EOB) symbol can be signaled earlier if trailing bands are
   all zero

The "cost" is implicitly minimized by sorting coefficient positions by their
zero count (ascending), so positions with fewer zeros (higher information
content) come first.

### Zero Count Quantization Formula

```
quantized_count = floor(raw_zero_count * (1.0 / sqrt(block_size)) + 0.1)
```

Where `block_size = kDCTBlockSize * covered_blocks_x * covered_blocks_y`.

The `1/sqrt(N)` normalization makes counts comparable across different block
sizes. The `+ 0.1` bias prevents values just below integer boundaries from
rounding down incorrectly.

The quantization intentionally coarsens the sort key so that coefficient
positions with similar zero counts preserve their natural (zig-zag) relative
ordering, reducing the Lehmer code cost. A finer sort key would produce a more
"optimal" scan order but would cost more bits to encode the permutation itself.

### Lehmer Code Encoding Cost

The permutation encoding cost is proportional to the number of non-zero Lehmer
code values. The `TokenizePermutation` function trims trailing zeros, and each
non-zero Lehmer value is entropy-coded using ANS with context derived from the
previous value. This means:

- Identity permutation (natural order) costs essentially nothing (just the
  length token = 0)
- Small perturbations of the natural order cost few bits
- Random permutations cost the most

The decision to use a custom order vs. default is implicit: if the computed
order matches the natural order for all 3 channels, the order bucket is removed
from `current_used_orders` (enc_coeff_order.cc:231-233), and the permutation is
not transmitted.

## Algorithm Details

### Speed Tier Gating

| Speed Tier | Numeric | Custom Orders? | Block Sampling | Max Customized Size |
|------------|---------|----------------|----------------|---------------------|
| kTectonicPlate | -1 | Yes | None | 32x32 (order <= 6) |
| kGlacier | 0 | Yes | None | 32x32 |
| kTortoise | 1 | Yes | None | 32x32 |
| kKitten | 2 | Yes | None | 32x32 |
| kSquirrel | 3 | Yes | 50% if DCT8-only | 32x32 |
| kWombat | 4 | Yes | 50% if DCT8-only | 32x32 |
| kHare | 5 | Yes | 50% if DCT8-only | 32x32 |
| kCheetah | 6 | Yes | 50% if DCT8-only | 32x32 |
| kFalcon | 7 | No (default only) | N/A | N/A |
| kThunder | 8 | No (default only) | N/A | N/A |
| kLightning | 9 | No (default only) | N/A | N/A |

At `speed >= kFalcon`, `ComputeUsedOrders` returns `{1, 1}` which marks only
DCT8 as "used" and "customizable". However, since `ComputeCoeffOrder` is gated
on `current_used_orders != 0`, and the actual optimization loop requires blocks
to exist for the given strategy, the practical effect at kFalcon+ is that only
default (natural) orders are used.

### Natural Order Generation (Generalized Zig-Zag)

For a block of size `(cy * 8) x (cx * 8)` where `cx >= cy`:

1. Compute the aspect ratio `xs = cx / cy`
2. Generate a zig-zag for a `(cx * 8) x (cx * 8)` square
3. Keep only rows where `y % xs == 0`, then compress `y` by dividing by `xs`
4. The first `cx * cy` entries (the LLF positions) are the top-left rectangle
5. Remaining entries follow the diagonal traversal pattern with alternating
   direction on odd/even diagonals (the classic zig-zag characteristic)

For the standard 8x8 DCT, this produces the traditional JPEG zig-zag order.
For non-square blocks like 16x8, it produces an order adapted to the
rectangular frequency grid.

### Lehmer Code Encoding/Decoding

The Lehmer code (factorial number system) encodes a permutation of N elements
as a sequence of N values where `lehmer[i]` is the number of elements less
than `permutation[i]` that appear after position `i` in the permutation.

**Encoding** (`ComputeLehmerCode`, lehmer_code.h:31-56):
- Uses a Fenwick tree to track which elements have been "used"
- For each position, computes the number of previously-seen smaller elements
  (the "penalty") via prefix sum, then `lehmer[i] = permutation[i] - penalty`
- O(N log N) time

**Decoding** (`DecodeLehmerCode`, lehmer_code.h:61-100):
- Uses an implicit order-statistics tree built on a Fenwick tree
- For each Lehmer code value, finds the k-th unused element via binary search
  on the Fenwick tree prefix sums
- O(N log^2 N) time (log N per element, N elements)

**Bitstream format:**
1. First token: effective permutation length minus skip (trailing identity
   elements are trimmed)
2. Subsequent tokens: Lehmer code values, each coded with context from previous
   value
3. Context selection: `CoeffOrderContext(val)` maps through
   `HybridUintConfig(0,0,0)` which produces `floor(log2(val))` for `val >= 1`,
   clamped to `[0, 7]`

### Permutation Application

On the encoder side (`EncodeCoeffOrder`):
1. The optimized order maps index -> coefficient position
2. Before encoding, convert to natural-order-relative permutation using
   `ComputeNaturalCoeffOrderLut`: `order_zigzag[i] = lut[order[i]]`
3. Encode this relative permutation as Lehmer code

On the decoder side (`DecodeCoeffOrder`):
1. Decode Lehmer code to get a permutation (relative to natural order)
2. Map through natural order: `order[k] = natural_order[permutation[k]]`

This means the bitstream encodes the *deviation* from natural order, not
absolute positions. A natural-order scan costs zero bits to encode.

### Multi-Pass Order Accumulation

`ComputeCoeffOrder` accepts `prev_used_acs` and `all_used_orders` to support
progressive/multi-pass encoding:
- `prev_used_acs`: AC strategies that were committed in previous passes (their
  orders are already fixed)
- `all_used_orders`: accumulator updated by the function; tracks which orders
  have been customized across all passes
- `current_used_acs`: strategies present in the current pass
- `current_used_orders`: which orders the current pass wants to customize

An order bucket is skipped if it was already committed in a previous pass
(`prev_used_acs` or `all_used_orders` bits are set).

### LLF Skip

When encoding/decoding permutations for multi-block transforms, the first
`llf = covered_blocks_x * covered_blocks_y` positions are skipped (the `skip`
parameter). These LLF coefficients always come first and are never permuted.
This is enforced both by setting `num_zeros = -1` for LLF positions during
optimization and by passing `skip = llf` to the permutation codec.

## Dependencies

### Upstream (this system depends on)
- `ac_strategy.h/cc` -- `AcStrategy`, `AcStrategyImage`, `AcStrategyType`,
  `ComputeNaturalCoeffOrder`, `ComputeNaturalCoeffOrderLut`, `kStrategyOrder`
- `frame_dimensions.h` -- `kBlockDim`, `kDCTBlockSize`, `kGroupDim`,
  `kGroupDimInBlocks`, `FrameDimensions`
- `lehmer_code.h` -- `ComputeLehmerCode`, `DecodeLehmerCode`, `LehmerT`
- `dec_ans.h` -- `HybridUintConfig`, `ANSSymbolReader`, `ANSCode`
- `enc_ans.h` -- `BuildAndEncodeHistograms`, `WriteTokens`, `Token`
- `enc_bit_writer.h` -- `BitWriter`
- `dec_bit_reader.h` -- `BitReader`
- `common.h` -- `SpeedTier`
- `dct_util.h` -- `ACImage`, `ACType`, `ConstACPtr`

### Downstream (depends on this system)
- AC coefficient encoding/decoding -- uses the order to serialize/deserialize
  coefficients in the optimized scan order
- Frame encoding pipeline -- calls `ComputeUsedOrders` and `ComputeCoeffOrder`
  as part of the encoding loop
- Bitstream header -- `EncodeCoeffOrders`/`DecodeCoeffOrders` are called during
  frame header serialization

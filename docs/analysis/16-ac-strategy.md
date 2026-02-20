# AC Strategy (Block Size Selection)

This is one of the most critical encoder decision systems. It selects which DCT transform
size to use for each 8×8 block region, directly controlling rate-distortion tradeoffs.

## Source Files
- `lib/jxl/ac_strategy.h` (275 lines) — AcStrategy type enum, AcStrategyImage, block size LUTs
- `lib/jxl/enc_ac_strategy.h` (91 lines) — ACSConfig struct, AcStrategyHeuristics interface
- `lib/jxl/enc_ac_strategy.cc` (1200 lines) — All heuristic logic, cost functions, merging

## Key Types

### `AcStrategyType` (enum, 27 values)
```
DCT=0, IDENTITY=1, DCT2X2=2, DCT4X4=3, DCT16X16=4, DCT32X32=5,
DCT16X8=6, DCT8X16=7, DCT32X8=8, DCT8X32=9, DCT32X16=10, DCT16X32=11,
DCT4X8=12, DCT8X4=13, AFV0=14, AFV1=15, AFV2=16, AFV3=17,
DCT64X64=18, DCT64X32=19, DCT32X64=20,
DCT128X128=21, DCT128X64=22, DCT64X128=23,
DCT256X256=24, DCT256X128=25, DCT128X256=26
```

### `AcStrategy` — block size descriptor
- `kMaxCoeffBlocks = 32` → max 32×32 blocks = 256×256 pixels
- `kNumValidStrategies = 27`
- Storage: `uint8_t` per 8×8 block. High 7 bits = strategy type, bit 0 = is_first_block flag
- `covered_blocks_x/y()` LUTs:
  - DCT8: 1×1, DCT16X16: 2×2, DCT32X32: 4×4, DCT64X64: 8×8
  - DCT16X8: 1×2, DCT8X16: 2×1, DCT32X8: 1×4, etc.

### `ACSConfig` — configuration passed to cost functions
```cpp
struct ACSConfig {
  const DequantMatrices* dequant;        // quantization matrices
  const float* quant_field_row;          // per-block quantization level
  const float* masking_field_row;        // per-block masking value
  const float* masking1x1_field_row;     // per-pixel masking value
  const float* src_rows[3];             // XYB source pixels
  float info_loss_multiplier;            // weight for distortion term
  float cost_delta;                      // weight for rate (entropy) term
  float zeros_mul;                       // weight for zero-count encoding cost
};
```

## Constants — Cost Function Parameters

### Rate-Distortion Balance (set in `AcStrategyHeuristics::Init`)
```
info_loss_multiplier = 1.2    (base, controls distortion weight)
zeros_mul = 9.3089059022677905  (base, controls zero-count cost)
cost_delta = 10.833273317067883  (base, controls per-coeff entropy cost)
```

These are **adjusted by butteraugli distance** via power functions:
```
kBias = 0.13731742964354549
ratio = (butteraugli_distance + kBias) / (1.0 + kBias)

info_loss_multiplier *= pow(ratio, 0.33677806662454718)
zeros_mul           *= pow(ratio, 0.50990926717963703)
cost_delta          *= pow(ratio, 0.36702940662370243)
```

At high quality (low butteraugli_distance), `ratio` < 1 → all multipliers decrease →
cost function favors rate reduction. At low quality, multipliers increase → favors
minimizing distortion.

### 8×8 Transform Entropy Multipliers
Each 8×8 candidate has an `entropy_mul` that biases selection:
```
DCT8×8:    0.8      (speed_tier ≤ 9)
DCT4×4:    1.08     (speed_tier ≤ 5)
DCT2×2:    0.95     (speed_tier ≤ 5)
DCT4×8:    0.85931637428340035  (speed_tier ≤ 4)
DCT8×4:    0.85931637428340035  (speed_tier ≤ 4)
IDENTITY:  1.0427542510634957  (speed_tier ≤ 5)
AFV0-3:    0.81779489591359944  (speed_tier ≤ 4)
```

The actual multiplier used is `tx.entropy_mul / DCT8_entropy_mul`, so DCT8 has multiplier 1.0,
DCT4×4 has 1.08/0.8 = 1.35, etc.

**Quality-dependent adjustments to 8×8 selection:**
- DCT2×2 and IDENTITY at high quality (target < 5.0):
  ```
  kFavor2X2AtHighQuality = 0.4
  weight = pow((5.0 - target) / 5.0, 2.0)
  entropy_mul -= 0.4 * weight
  ```
  Makes small transforms cheaper at high quality.

- Non-standard transforms at low quality (target > 4.0):
  ```
  kAvoidEntropyOfTransforms = 0.5
  mul = 1.0   (if target >= 12.0)
  mul = (12.0 - 4.0) / (target - 4.0)  (if 4.0 < target < 12.0)
  entropy_mul += 0.5 * mul
  ```
  Penalizes exotic transforms at low quality.

### Merge Entropy Multipliers (larger-than-8×8 transforms)
```
entropy_mul16X8  = 1.21    (DCT16X8, DCT8X16)
entropy_mul16X16 = 1.34    (DCT16X16)
entropy_mul16X32 = 1.49    (DCT16X32, DCT32X16)
entropy_mul32X32 = 1.48    (DCT32X32)
entropy_mul64X32 = 2.25    (DCT64X32, DCT32X64)
entropy_mul64X64 = 2.25    (DCT64X64)
```

These multipliers are >1.0, making larger transforms more expensive. This prevents
ringing artifacts — the comment says "Optimization will find smaller numbers and produce
more ringing than is ideal."

### 8×8 Favor Multiplier
```
k8x8mul1 = -0.4
k8x8mul2 = 1.0
k8x8base = 1.4
mul8x8 = k8x8mul2 + k8x8mul1 / (butteraugli_target + k8x8base)
```
At high quality (target≈1): mul8x8 ≈ 1.0 + (-0.4)/(2.4) ≈ 0.833
At low quality (target≈10): mul8x8 ≈ 1.0 + (-0.4)/(11.4) ≈ 0.965

8×8 blocks get a ~5-17% entropy discount, favoring smaller blocks.

## Cost Functions — DETAILED

### `EstimateEntropy()` — The Core Cost Function
`enc_ac_strategy.cc:364-511`

This estimates the total cost (rate + distortion) for a given transform applied to a block:

**Step 1: Forward transform**
For each channel c ∈ {X, Y, B}: apply the DCT transform to source pixels.

**Step 2: Compute quantization norm**
The quant norm aggregates per-block quantization values:
- 1 block: `quant_norm16 = Quant(bx, by)` (direct value)
- 2 blocks: `quant_norm16 = max(Quant(bx,by), Quant(bx2,by2))` ("Taking max instead of 8th norm seems to work better for smallest blocks" —Jyrki)
- 4+ blocks: **16th-norm** across all sub-blocks:
  ```
  for each sub-block (ix, iy):
    qval = Quant(bx+ix, by+iy)
    quant_norm16 += qval^16    // qval*qval*qval*...*qval (8 muls = 16th power)
  quant_norm16 /= num_blocks
  quant_norm16 = quant_norm16^(1/16)
  ```

**Step 3: Compute entropy (rate estimate)**
For each channel c:
```
for each coefficient i:
  val = (coeff[i] - Y_coeff[i] * cmap_factor[c]) * inv_quant_matrix[i] * quant_norm16
  rval = round(val)
  q = abs(rval)

  entropy += sqrt(q)      // NOTE: sqrt, not q*C — "punishing large values less aggressively"
  nzeros += (q != 0)

entropy_channel = cost_delta * sum(entropy)
                + zeros_mul * (CeilLog2(nbits + 17) + nbits)
                  where nbits = CeilLog2(nzeros + 1) + 1
```

The `sqrt(q)` entropy model is notable — it replaced `q * C` because "that cost model
seems to be punishing large values more than necessary."

**Step 4: X-channel penalty for large blocks**
```
if c == 0 (X channel) and num_blocks >= 2:
  w = 1.0 + min(3.0, num_blocks / 8.0)
  entropy *= w
  loss *= w
```
Large blocks in X (red-green) channel get penalized because "we often see ringing."
At num_blocks=8 (DCT64X64): w = 2.0 (double the cost).

**Step 5: Compute distortion (information loss)**
For each channel c:
```
// Reconstruct quantization error in pixel domain
diff_coeffs[i] = quant_matrix[i] * (val - round(val))
inverse_transform(diff_coeffs) → error_pixels

// Compute weighted 8th-power norm of error
masku_lut = {12.0, 0.0, 4.0}  // per-channel masking offsets: X=12, Y=0, B=4
for each pixel:
  masked_error = (masking1x1[pixel] + masku_lut[c]) * error_pixel
  lossc += masked_error^8

kChannelMul = { 8.2^8, 1.0^8, 1.03^8 }   // X≈2.09e7, Y=1.0, B≈1.27
lossc *= kChannelMul[c]
loss += lossc
```

**Step 6: Combine rate + distortion**
```
loss_scalar = (sum(loss) / (num_blocks * 64))^(1/8) * (num_blocks * 64) / quant_norm16
entropy *= entropy_mul      // the per-transform bias multiplier
total_cost = entropy + info_loss_multiplier * loss_scalar
```

## Decision Tree — Block Size Selection

### Phase 1: Best 8×8 transform per block (`FindBest8x8Transform`)
`enc_ac_strategy.cc:513-613`

For each 8×8 block in the 64×64 region:
1. Try all 10 single-block transforms (gated by speed_tier)
2. Evaluate `EstimateEntropy()` for each with their biased entropy_mul
3. Select minimum cost

### Phase 2: Hierarchical merging (`ProcessRectACS`)
`enc_ac_strategy.cc:827-1059`

Works in a 64×64 block (8×8 in block coordinates). Steps:

1. **Speed gate**: If speed_tier > Hare, return (8×8 only)

2. **8×8 selection** for all 64 sub-blocks, multiply by `mul8x8` discount

3. **Priority-based rectangular merging** (`kTransformsForMerge`):
   ```
   Priority 2: DCT16X8/DCT8X16  (decoding_speed ≤ 4, encoding_speed ≤ 5)
   Priority 4: DCT16X32/DCT32X16 (decoding_speed ≤ 4, encoding_speed ≤ 4)
   Priority 6: DCT64X32/DCT32X64 (decoding_speed ≤ 1, encoding_speed ≤ 3)
   ```

4. **Square transform merging** via `FindBestFirstLevelDivisionForSquare()`:
   - Called for 2×2, 4×4, and 8×8 block regions
   - For each region, compare 3 options:
     - Square transform (e.g., DCT16X16 for 2×2 region)
     - Two vertical rectangles (e.g., 2× DCT16X8)
     - Two horizontal rectangles (e.g., 2× DCT8X16)
   - Decision:
     ```
     costJxN = min(entropy_JXK_left, entropy[0][0]+entropy[1][0])
             + min(entropy_JXK_right, entropy[0][1]+entropy[1][1])
     costNxJ = min(entropy_KXJ_top, entropy[0][0]+entropy[0][1])
             + min(entropy_KXJ_bottom, entropy[1][0]+entropy[1][1])

     if entropy_JXJ < costJxN and entropy_JXJ < costNxJ:
       → use square transform
     elif costJxN < costNxJ:
       → use vertical split where beneficial
     else:
       → use horizontal split where beneficial
     ```

5. **Non-aligned merging** (speed_tier < Hare):
   - Retry 16×16 merging at non-2-aligned positions
   - Retry 32×32 merging at non-4-aligned positions (step=2 for Tortoise, step=1 otherwise)

### Speed Tier Gating Summary
```
Cheetah (≥7): DCT8 only, no heuristics at all
Hare (5-6):   8×8 selection only (10 transforms), no merging
Wombat (4):   + 16×8/8×16 merge, + 16×32/32×16 merge
Squirrel (3): + 64×32/32×64 merge
Kitten (2):   + non-aligned 32×32 merging (step=2)
Tortoise (1): + non-aligned 32×32 merging (step=1)
```

## Algorithm Overview — Merging with Priority
`TryMergeAcs()` at `enc_ac_strategy.cc:618-653`:

1. Check if any sub-block already has higher priority → abort (prevents overlaps)
2. Sum current entropy estimates for all sub-blocks
3. Estimate entropy for the candidate merged transform
4. If candidate < current sum → accept merge, update priority map and entropy estimates

The priority system (uint8_t per 8×8 block) prevents incompatible overlapping transforms
(e.g., DCT64X32 vs DCT32X64 at the same location).

## Dependencies
- **Depends on**: DequantMatrices (quant_weights), ColorCorrelationMap (cmap), ImageF (mask/quant fields),
  TransformFromPixels/TransformToPixels (DCT), CompressParams (speed_tier, butteraugli_distance)
- **Depended on by**: enc_heuristics.cc (LossyFrameHeuristics calls AcStrategyHeuristics)

## Mermaid Diagram Data

```
Main flow:
  Input: XYB pixels, quant_field, mask, mask1x1
  → For each 64×64 block:
    → Phase 1: Best 8×8 for each sub-block (10 candidates)
    → Phase 2: Try 16×8 merges (priority 2)
    → Phase 2: Try 16×16 via FindBestFirstLevelDivisionForSquare
    → Phase 2: Try 16×32 merges (priority 4)
    → Phase 2: Try 32×32 via FindBestFirstLevelDivisionForSquare
    → Phase 2: Try 64×32 merges (priority 6)
    → Phase 2: Try 64×64 via FindBestFirstLevelDivisionForSquare
    → Phase 3: Non-aligned 16×16 and 32×32 merges
  Output: AcStrategyImage
```

## Open Questions
- DCT128×128, DCT256×256 strategies exist in the enum but are NOT used in merge heuristics
- DCT32X8 and DCT8X32 also not used in merge heuristics (commented out with entropy_mul 2.26)
- The `kChannelMul` distortion weights (8.2^8 for X) seem very strong — how were they tuned?
- The comment says constants were optimized with `simplex_fork.py` (Nelder-Mead variant)
  minimizing BPP * pnorm, then manually adjusted by Jyrki for visual quality

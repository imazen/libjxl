# Adaptive Quantization

The AQ system computes a per-block quantization field that allocates more bits to
perceptually sensitive areas and fewer bits to areas where the eye is less sensitive.
It includes a multi-iteration butteraugli feedback loop for quality refinement.

## Source Files
- `lib/jxl/enc_adaptive_quantization.h` (58 lines) — public API declarations
- `lib/jxl/enc_adaptive_quantization.cc` (1292 lines) — all masking, AQ map, feedback loop

## Key Types

### `AdaptiveQuantizationImpl`
```cpp
struct AdaptiveQuantizationImpl {
  std::vector<ImageF> pre_erosion;  // per-thread scratch (2x downsampled)
  ImageF aq_map;                     // output: per-block AQ values
  ImageF diff_buffer;                // per-thread row buffer
};
```

## Constants — Masking and Modulation

### ComputeMask — Base masking function
`enc_adaptive_quantization.cc:95-117`

Converts a raw AQ value to a masking exponent via rational function:
```
kBase = -0.7647
kMul0 = 0.80061762862741759
kMul2 = 17.35036561631863
kMul3 = 6.7943250517376494
kMul4 = 9.4708735624378946
kOffset2 = 302.59587815579727
kOffset3 = 3.7179635626140772
kOffset4 = 0.25 * kOffset3

v1 = max(out_val * kMul0, 1e-3)
mask = kBase + kMul4/(v1²+kOffset4) + kMul2/(v1+kOffset2) + kMul3/(v1²+kOffset3)
```

This produces higher masking (= lower quantization) in smooth areas and lower masking
in detailed areas.

### SimpleGamma Ratio — Opsin-to-Butteraugli space conversion
`enc_adaptive_quantization.cc:120-153`

Links jxl's cubic-root opsin space to butteraugli's log-gamma perceptual space:
```
kSGmul = 226.77216153508914
kSGmul2 = 1.0 / 73.377132366608819
kSGRetMul = kSGmul2 * 18.6580932135 * kInvLog2e
kSGVOffset = 7.7825991679894591

RatioOfDerivativesOfCubicRootToSimpleGamma(v):
  v = max(v, 0)
  num = kSGRetMul * 3 * kSGmul * v²  + epsilon
  den = kInvLog2e * kSGmul * v³ + kSGVOffset * kInvLog2e + epsilon
  return den/num   (or num/den if inverted)
```

The XYB gamma is 3.0 (cubic root) for fast decoding. Butteraugli's perceptual gamma
is ~2.6. This ratio function bridges between them.

### GammaModulation — Luminance-dependent masking adjustment
`enc_adaptive_quantization.cc:179-211`

For each 8×8 block:
```
for each pixel (x, y) in block:
  iny = xyb_y[x,y] + 0.16  (kBias, ensures positive)
  inx = xyb_x[x,y]
  r = iny - inx   (red channel in LMS-like space)
  g = iny + inx   (green channel in LMS-like space)
  overall_ratio += RatioOfDerivatives(r) + RatioOfDerivatives(g)

overall_ratio *= 0.5 / 64  (average over block, two channels)
kGamma = 0.1005613337192697
out_val += kGamma * FastLog2(overall_ratio)
```

This adjusts quantization based on the local luminance — darker areas get different
treatment because the eye's contrast sensitivity varies with adaptation level.

### HfModulation — High-frequency content adjustment
`enc_adaptive_quantization.cc:259-313`

Measures local pixel variation using 4-connected differences:
```
valmin_y = 0.0206  (clamp threshold — ignores tiny differences)
kMul_y = -0.38     (negative = high-frequency content REDUCES quantization)
kOffset = 0.42

for each pixel in 8×8 block:
  sum_y += min(valmin_y, |pixel - right_neighbor|)
  sum_y += min(valmin_y, |pixel - bottom_neighbor|)

out_val += kMul_y * sum_y + kOffset
```

High-frequency areas get a negative adjustment (higher value = coarser quant), letting
the encoder spend fewer bits on already-detailed areas.

### BlueModulation — S-cone receptor compensation
`enc_adaptive_quantization.cc:220-256`

Increases precision where blue content dominates and M/L cone activations are low:
```
kLimit = 0.010474084867598155
kOffset = 0.0031994768654636393
kMul = 0.90590804735610064
kMaxLimit = 15.463398341612438

for each pixel in 8×8 block:
  p_y_effective = p_y + kOffset + |p_x|
  if p_b > p_y_effective:
    sum += min(p_b - p_y_effective, kLimit)

// Anti-all-blue: if entire block is blue, DON'T boost
if sum >= 32 * kLimit:
  sum = 64 * kLimit - sum  (mirror: fully blue = no boost)
sum = min(sum, kMaxLimit * kLimit)
out_val += kMul * sum
```

Rationale: when M and L cones are inactive (dark, saturated blue), S cones become
dominant for spatial vision → need more bits.

### PerBlockModulations — Final assembly
`enc_adaptive_quantization.cc:315-348`

Combines all modulations into the AQ map:
```
base_level = 0.48 * scale
kDampenRampStart = 2.0
kDampenRampEnd = 14.0

if butteraugli_target >= 2.0:
  dampen = 1.0 - (target - 2.0) / (14.0 - 2.0)
  dampen = max(dampen, 0)

mul = scale * dampen
add = (1.0 - dampen) * base_level

for each block:
  mask_val = ComputeMask(aq_raw_value)
  mask_val = GammaModulation(mask_val)
  out = HfModulation(mask_val)
  out = min(out, BlueModulation(mask_val))  // Blue only reduces (more precise)
  aq_map[block] = pow(2, out * 1.442695041) * mul + add
```

Key: the modulations work in log-space (exponents), then `2^x` converts to a
multiplicative quantization field. At high distance (>14), dampen=0 and AQ map
becomes a flat `base_level`.

### MaskingSqrt — For pre-erosion
```
kLogOffset = 27.505837037000106
kMul = 211.66567973503678
MaskingSqrt(v) = 0.25 * sqrt(v * sqrt(kMul * 1e8) + kLogOffset)
```

## Cost Functions — Butteraugli Feedback Loop

### InitialQuantDC
`enc_adaptive_quantization.cc:1250-1262`
```
kDcQuantPow = 0.83
kDcQuant = 1.095924047623553
kDcMul = 0.3

butteraugli_target_dc = max(0.5 * target, min(target, 0.3 * pow(target/0.3, 0.83)))
dc_quant = min(kDcQuant / butteraugli_target_dc, 50.0)
```

### InitialQuantField
`enc_adaptive_quantization.cc:1264-1271`
```
kAcQuant = 0.765
quant_ac = kAcQuant / butteraugli_target
→ calls AdaptiveQuantizationMap(target, opsin, rect, quant_ac * rescale, ...)
```

### AdjustQuantField — Per-AC-strategy quant unification
`enc_adaptive_quantization.cc:1198-1248`

For multi-block transforms, replaces per-8×8 quant values with a single value:
```
kLimit = 1.54138
kMul = 0.56391

mean_max_mixer = 1.0
if target > kLimit:
  mean_max_mixer = max(0, 1.0 - (target - kLimit) * kMul)

for each multi-block transform (≥4 sub-blocks):
  max_q = max of sub-block quant values
  mean_q = mean of sub-block quant values
  unified_q = mean_max_mixer * max_q + (1 - mean_max_mixer) * mean_q
```

At low quality (high target): uses mean (avoid over-quantizing).
At high quality (low target): uses max (preserve worst-case block).

### FindBestQuantization — Main iterative loop
`enc_adaptive_quantization.cc:929-1115`

The core quality refinement loop. **2-4 iterations** (2 default, 4 for Tortoise):

```
kDefaultButteraugliIters = 2
kMaxButteraugliIters = 4
kOriginalComparisonRound = 1

For each iteration i:
  1. Set quantizer from quant_field
  2. Encode and decode image (RoundtripImage)
  3. Compare decoded vs reference using butteraugli → score, diffmap
  4. Compute tile_distmap (16th-norm over AC-strategy-aligned tiles)
  5. Adjust quant_field based on distortion

Iteration 0-1: kPow = {0.2, 0.2}
  For tiles where distortion > target:
    quant_field *= diff   (increase quantization to reduce bits)
    if rounding unchanged: quant_field += quantizer.Scale()  (force at least 1 step)
  For tiles where distortion ≤ target:
    quant_field *= pow(diff, 0.2)  (gently reduce quantization)

Iteration 1 (kOriginalComparisonRound):
  Apply clamping to prevent drift from initial guess:
    kInitMul = 0.6
    clamp = 0.4 * quant + 0.6 * initial_quant
    quant = max(quant, clamp)

Iterations 2+: kPow = 0.0
  Only adjust tiles exceeding target (no quality improvement).
```

### TileDistMap — Distortion aggregation
`enc_adaptive_quantization.cc:768-833`

Computes per-AC-strategy-block distortion using 16th-norm:
```
kBorderMul = 0.98
kCornerMul = 0.7
kTileNorm = 1.2

for each tile (aligned to AC strategy blocks):
  for each pixel in tile + margin:
    xmul = border/corner weighting
    dist_norm += xmul * row[x]^16
    pixels += xmul
  tile_dist = kTileNorm * (dist_norm / pixels)^(1/16)
```

The 16th norm is between max and mean — it strongly weights the worst pixels but
isn't as extreme as pure max.

## Decision Tree — Quality Mode Selection

### FindBestQuantizer — Mode dispatch
`enc_adaptive_quantization.cc:1273-1289`
```
if max_error_mode:
  → FindBestQuantizationMaxError (per-block max error targeting)
elif linear != null AND speed_tier <= Kitten:
  → FindBestQuantization (butteraugli iterative loop)
else:
  → no refinement (initial quant field used as-is)
```

Speed tiers Hare and above skip the butteraugli feedback loop entirely.

### ComputeTile — AQ Map Generation Pipeline
`enc_adaptive_quantization.cc:466-628`

```
1. Compute 1×1 masking (per-pixel Laplacian of Y channel):
   base = average of 4 neighbors
   gammac = RatioOfDerivatives(pixel + 0.019)
   diff = |gammac * (pixel - base)|
   diff = log1p(diff)
   mask1x1[x,y] = 1.0 / (diff + 0.01)

2. Compute 4×4-subsampled differences:
   Same Laplacian but squared, clamped to 0.2, then MaskingSqrt
   Subsample 4×4 → pre_erosion buffer

3. FuzzyErosion (2× downsampling with weighted 4-smallest):
   3×3 neighborhood → find 4 smallest values
   kMulBase = {0.125, 0.1, 0.09, 0.06}
   kMulAdd = {0.0, -0.1, -0.09, -0.06}  (quality-dependent)
   kTotal = 0.29959705784054957
   output = weighted sum of 4 smallest, normalized

4. ComputeMaskForAcStrategyUse:
   mask[block] = 1.0 / (aq_map[block] + 0.001)

5. PerBlockModulations (combines all modulations via exponentiation)

6. Blur1x1Masking (Symmetric5 convolution on mask1x1):
   kFilterMask1x1 = {0.364911248, 0.05, 0.1688888021, 0.221069183, 0.306563504}
```

## Dependencies
- **Depends on**: butteraugli (comparator), Image3F (XYB), CompressParams,
  AcStrategyImage, Quantizer, enc_group (DecodeGroupForRoundtrip), RenderPipeline
- **Depended on by**: enc_heuristics.cc (LossyFrameHeuristics calls InitialQuantField,
  FindBestQuantizer)

## Mermaid Diagram Data
```
AQ Pipeline:
  XYB Image → 1×1 Laplacian → per-pixel masking (mask1x1)
                             → 4×4 subsample → FuzzyErosion → base AQ map
  Base AQ map → ComputeMask → GammaModulation → HfModulation → BlueModulation
             → exp2() → aq_map (per-block quantization field)

Quality Refinement Loop:
  aq_map → Quantizer → Encode → Decode → Butteraugli Compare
        → TileDistMap (16th norm) → Adjust quant_field → repeat (2-4×)
```

## Open Questions
- The `match_gamma_offset = 0.019` bridges the XYB cubic root (gamma 3.0) to butteraugli's
  ~2.6 gamma. Why 0.019 specifically? (Empirically tuned?)
- FuzzyErosion uses the 4 smallest of 9 neighbors — this is a soft morphological erosion.
  How sensitive is quality to the kMulBase/kMulAdd weights?
- The kFilterMask1x1 kernel (Symmetric5) has unusual weights. Were they optimized?

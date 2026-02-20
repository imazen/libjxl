# Encoder Features (CfL, Noise, Patches, Splines)

## Chroma-from-Luma (CfL)

### Source Files

- `lib/jxl/chroma_from_luma.h` -- Decoder-side types: `ColorCorrelation`, `ColorCorrelationMap`
- `lib/jxl/chroma_from_luma.cc` -- Decoder-side: `DecodeDC`, `ColorCorrelationMap::Create`
- `lib/jxl/enc_chroma_from_luma.h` -- Encoder-side: `CfLHeuristics` struct
- `lib/jxl/enc_chroma_from_luma.cc` -- Encoder-side: `CFLFunction`, `FindBestMultiplier`, `ComputeTile`
- `lib/jxl/cms/opsin_params.h` -- Opsin color space constants including `kYToBRatio`
- `lib/jxl/enc_heuristics.cc` -- Invokes CfL tile computation during encoding

### Key Types

**`ColorCorrelation`** (in `chroma_from_luma.h`):
Global CfL parameters shared by encoder and decoder. Stores the linear model
that predicts X and B channels from Y.

Fields:
- `color_factor_` (uint32_t, default `kDefaultColorFactor` = 84) -- denominator for the per-tile integer factor
- `color_scale_` (float, = `1.0f / color_factor_`) -- precomputed reciprocal
- `base_correlation_x_` (float, default 0.0) -- global X-from-Y base offset
- `base_correlation_b_` (float, default `kYToBRatio` = 1.0) -- global B-from-Y base offset
- `ytox_dc_` (int32_t, default 0) -- DC-level X-from-Y integer factor
- `ytob_dc_` (int32_t, default 0) -- DC-level B-from-Y integer factor
- `dc_factors_[4]` (float) -- precomputed DC ratios: `[0]` = YtoXRatio(ytox_dc_), `[2]` = YtoBRatio(ytob_dc_)

Conversion functions:
```
YtoXRatio(x_factor) = base_correlation_x_ + x_factor * color_scale_
YtoBRatio(b_factor) = base_correlation_b_ + b_factor * color_scale_
```

So with defaults: `YtoXRatio(x_factor) = 0.0 + x_factor / 84`, and
`YtoBRatio(b_factor) = 1.0 + b_factor / 84`.

**`ColorCorrelationMap`** (in `chroma_from_luma.h`):
Per-tile CfL maps stored as signed byte images. Each tile is 64x64 pixels
(8x8 blocks).

Fields:
- `ytox_map` (ImageSB) -- per-tile X-from-Y integer factors (int8_t, range -128..127)
- `ytob_map` (ImageSB) -- per-tile B-from-Y integer factors (int8_t, range -128..127)
- `base_` (ColorCorrelation) -- the global parameters

The map dimensions are `ceil(xsize / 64)` by `ceil(ysize / 64)`.

**`CfLHeuristics`** (in `enc_chroma_from_luma.h`):
Encoder-side working state for CfL parameter estimation. Manages per-thread
scratch memory for DCT blocks and coefficient buffers.

**`CFLFunction`** (in `enc_chroma_from_luma.cc`, inside HWY namespace):
The cost function whose root is the optimal CfL multiplier.

### Constants (EXACT values)

| Constant | Value | Location |
|---|---|---|
| `kColorTileDim` | 64 (pixels) | `chroma_from_luma.h:28` |
| `kColorTileDimInBlocks` | 8 (= 64/8) | `chroma_from_luma.h:32` |
| `kDefaultColorFactor` | 84 | `chroma_from_luma.h:37` |
| `kCFLFixedPointPrecision` | 11 (bits) | `chroma_from_luma.h:43` |
| `kYToBRatio` | 1.0 | `cms/opsin_params.h:33` |
| `CFLFunction::kCoeff` | 1.0/3 | `enc_chroma_from_luma.cc:56` |
| `CFLFunction::kThres` | 100.0 | `enc_chroma_from_luma.cc:57` |
| `CFLFunction::kInvColorFactor` | 1.0/84 | `enc_chroma_from_luma.cc:58` |
| `kDistanceMultiplierAC` | 1e-9 | `enc_chroma_from_luma.cc:217` |
| `kStrangeMultiplier` | 128 | `enc_chroma_from_luma.cc:330` |
| `kEncTileDim` | 64 (pixels) | `enc_params.h:205` |
| `kEncTileDimInBlocks` | 8 (= 64/8) | `enc_params.h:206` |

### Algorithm Details

#### CfL Model

The CfL model removes chroma-luma correlation by predicting the X and B
channels from the Y channel using a linear relationship:

```
residual_x = X - Y * factor_x
residual_b = B - Y * factor_b
```

where `factor_x = base_correlation_x + x_factor / color_factor` and
`factor_b = base_correlation_b + b_factor / color_factor`.

With defaults (`base_correlation_x=0`, `base_correlation_b=1.0`,
`color_factor=84`), a per-tile `ytob_map` value of 0 means factor_b = 1.0
(identity correlation), and each integer step adjusts by 1/84 ~= 0.0119.

The per-tile factors are int8_t values in [-128, 127], giving a factor range
of approximately -1.52 to +1.51 relative to the base correlation.

#### Per-Tile AC Coefficient Processing (`ComputeTile`)

For each 64x64 tile, the encoder:

1. **Iterates over AC blocks**: For each block in the tile, determined by the
   AC strategy (block size), it performs a forward DCT on all three XYB planes.

2. **Extracts DC values**: `DCFromLowestFrequencies()` extracts DC from the
   transform coefficients. DC values are scaled by the quantizer DC step and
   stored separately (4 rows: Y-for-X, X, Y-for-B, B).

3. **Zeroes out LF coefficients**: The low-frequency coefficients within each
   block are set to zero so they do not affect the AC CfL optimization. This
   allows simpler SIMD loops without special-casing LF positions.

4. **Applies quantization weighting**: Each AC coefficient is multiplied by
   `quantizer_scale * kStrangeMultiplier(128) * raw_quant * inv_dequant_matrix`.
   This weights coefficients by their perceptual importance (inverse of the
   dequantization matrix = quantization weights). When use_dct8 is true
   (no AC strategy available), the quantization weight is 1.

5. **Accumulates weighted coefficients**: The weighted Y*qm_x and X*qm_x
   (for the X channel) and Y*qm_b and B*qm_b (for the B channel) are stored
   in flat arrays (`coeffs_yx`, `coeffs_x`, `coeffs_yb`, `coeffs_b`).

6. **Calls `FindBestMultiplier`**: Separately for X-from-Y (base=0.0) and
   B-from-Y (base=kYToBRatio=1.0), with `distance_mul = 1e-9`.

#### Cost Function (`CFLFunction`)

The cost function is:

```
f(x) = (1/3) * sum_i [ (|a_i * x + b_i| + 1)^2 - 1 ] + distance_mul * x^2 * num
```

where:
- `a_i = values_m[i] / color_factor` (weighted luma coefficient / 84)
- `b_i = base * values_m[i] - values_s[i]` (base prediction minus actual chroma)
- The `(|v|+1)^2 - 1` form is a smoothed absolute-value-like penalty that is
  differentiable at zero and grows quadratically
- The `distance_mul * x^2 * num` term is an L2 regularizer that biases toward
  x=0 (no correction), with `distance_mul = 1e-9`
- Terms where `|a_i*x + b_i| >= kThres(100)` are excluded (outlier rejection)

The `Compute` method returns the **first derivative** f'(x), not f(x) itself.
It also computes f'(x+eps) and f'(x-eps) for numerical second derivative
estimation. The first derivative of the smoothed cost is:

```
f'(x) = 2 * distance_mul * num * x + (2/3) * sum_i [ a_i * (|v_i| + 1) * sign(v_i) ]
```

where `v_i = a_i * x + b_i` and terms with `|v_i| >= 100` are excluded.

#### Newton Optimization (`FindBestMultiplier`, slow path)

When `fast=false`, the encoder uses up to 20 iterations of Newton's method
with numerically approximated second derivatives:

```
eps = 100
for each iteration:
    f'(x) = CFLFunction::Compute(x, eps, &f'(x+eps), &f'(x-eps))
    f''(x) ~= (f'(x+eps) - f'(x-eps)) / (2*eps)
    step = f'(x) / (f''(x) + 0.85)             // 0.85 = stabilizer
    x -= clamp(step, -20, 20)                   // clamp = 20.0
    if |step| < 3e-3: break
```

Key observations:
- The numerical derivative uses a **large epsilon of 100** because the exact
  derivatives are very noisy (the cost function has many near-discontinuities
  from the absolute value and the threshold cutoff at kThres=100).
- The stabilizer `0.85` prevents division by near-zero second derivative.
- The step is clamped to [-20, 20] to prevent wild oscillations.
- Starting point is x=0 (no CfL correction).

#### Fast Path (`FindBestMultiplier`, fast=true)

When `fast=true`, the encoder uses a closed-form least-squares solution that
ignores the smoothed absolute value and threshold, treating the cost as pure
quadratic:

```
minimize sum_i (a_i * x + b_i)^2 + distance_mul * x^2 * num / 2
```

Solution: `x = -sum(a_i * b_i) / (sum(a_i^2) + num * distance_mul * 0.5)`

This is a simple linear regression with L2 regularization.

#### Towards-Zero Shrinkage

After optimization, the result is shrunk toward zero by `towards_zero = 2.6`:

```
if x >= 2.6:   x -= 2.6
elif x <= -2.6: x += 2.6
else:           x = 0
```

This is a soft-thresholding operation. The comment says CfL is "tricky for
larger transforms for HF components close to zero" and this "reduces red-green
oscillations." A variance-based approach (applying shrinkage only in
high-variance regions) would give ~1% more compression density.

The final result is rounded and clamped to int8_t range [-128, 127].

#### Speed-Tier Dispatch

From `enc_heuristics.cc`:

- **Squirrel and slower** (speed_tier <= kSquirrel): CfL computed first with
  `use_dct8=true` (no AC strategy), no quantizer. This gives an initial CfL
  map before block size decisions.
- **Hare and slower** (speed_tier <= kHare): CfL recomputed with actual AC
  strategy and quantization field. Uses `fast=true` for Wombat speed, `fast=false`
  for Hare and slower.
- **Faster than Hare**: No per-tile CfL; the map stays all-zeros (using only
  the global base correlation).

#### DC CfL Encoding

The `ColorCorrelationEncodeDC` function writes the global CfL parameters. If
all are default (ytox_dc=0, ytob_dc=0, color_factor=84, base_x=0,
base_b=kYToBRatio), it writes a single 1-bit flag. Otherwise it writes:
- 1-bit flag (0 = non-default)
- color_factor via U32Coder with distribution: Val(84), Val(256), BitsOffset(8,2), BitsOffset(16,258)
- base_correlation_x as float16
- base_correlation_b as float16
- ytox_dc as uint8 (offset by int8_min = -128)
- ytob_dc as uint8 (offset by int8_min = -128)

#### JPEG Compatibility

`ColorCorrelation::IsJPEGCompatible()` returns true when base correlations are 0,
DC correlations are 0, and color_factor is 84. The `RatioJPEG` function converts
a CfL factor to fixed-point with 11-bit precision (kCFLFixedPointPrecision)
for JPEG DCT coefficient processing.


## Noise Synthesis

### Source Files

- `lib/jxl/noise.h` -- Shared types: `NoiseParams`, `NoiseLevel`, `IndexAndFrac`
- `lib/jxl/enc_noise.h` -- Encoder API: `GetNoiseParameter`, `EncodeNoise`
- `lib/jxl/enc_noise.cc` -- Full estimation pipeline
- `lib/jxl/enc_optimize.h` -- Scaled Conjugate Gradient optimizer

### Key Types

**`NoiseParams`** (in `noise.h`):
An 8-point lookup table mapping intensity to noise strength.
- `lut` -- `std::array<float, 8>` (kNumNoisePoints = 8)
- `HasAny()` -- returns true if any LUT entry exceeds 1e-3
- `Clear()` -- zeroes all entries

**`NoiseLevel`** (in `noise.h`):
A single observation: `{noise_level, intensity}` measured from a flat patch.

**`NoiseHistogram`** (in `enc_noise.cc`):
256-bin histogram of SAD (Sum of Absolute Differences) scores, used to find the
mode and separate flat patches from textured ones.

**`LossFunction`** (in `enc_noise.cc`):
The optimization target for fitting the noise curve. Implements the interface
expected by `OptimizeWithScaledConjugateGradientMethod`.

### Constants

| Constant | Value | Location |
|---|---|---|
| `kNoisePrecision` | 1024.0 (10 bits) | `noise.h:22` |
| `kNoiseLutMax` | 1023.4999/1024 ~= 0.9995 | `noise.h:23` |
| `kNumNoisePoints` | 8 | `noise.h:29` |
| `block_s` (encoder patch size) | 8 | `enc_noise.cc:348` |
| `kNumBin` | 256 | `enc_noise.cc:349` |
| SAD threshold upper limit | 0.15 | `enc_noise.cc:357` |
| `kReg` (regularization) | 0.005 | `enc_noise.cc:162` |
| `kAsym` (asymmetric weight) | 1.1 | `enc_noise.cc:163` |
| `kMaxError` | 1e-3 | `enc_noise.cc:201` |
| SCG precision | 1e-8 | `enc_noise.cc:202` |
| SCG max iterations | 40 | `enc_noise.cc:203` |
| quality_coef multiplier | 1.4 | `enc_noise.cc:364` |

### Algorithm Details

#### Overview

The noise estimation pipeline:
1. Divides the image into 8x8 blocks
2. Scores each block for "flatness" using patch-based SAD
3. Identifies flat blocks via the SAD histogram mode
4. Measures noise level in flat blocks using a Laplacian filter
5. Fits an 8-point intensity-to-noise curve using conjugate gradient optimization

#### Step 1: SAD Scoring (`GetScoreSumsOfAbsoluteDifferences`)

For each 8x8 block, the function computes SAD between every 3x4 sub-patch and
a center 3x4 patch (at offset 2,2). The input signal is `0.5*(X+Y)` from the
opsin representation. All sub-patch SADs are sorted, and the mean of the lower
half is returned (ROAD-like robust estimator). This gives a texture strength
score where lower = flatter.

#### Step 2: SAD Threshold (`GetSADThreshold`)

A 256-bin histogram of the SAD scores is built. The mode (most frequent bin)
is taken as the representative "flat" value. The threshold is `mode / 256`.

If the threshold is > 0.15 (strong texture/pattern) or <= 0.0, noise
estimation is abandoned (returns false, no noise parameters).

#### Step 3: Noise Level Measurement (`GetNoiseLevel`)

For each block whose SAD score is at or below the threshold (i.e., flat blocks):

1. **Mean intensity**: Average of `0.5*(X+Y)` over the 8x8 block.

2. **Noise level**: Apply a 3x3 Laplacian-like filter to each pixel:
   ```
   [-0.25, -1.0, -0.25]
   [-1.0,   5.0, -1.0 ]
   [-0.25, -1.0, -0.25]
   ```
   The mean absolute filtered value over the block is the noise level.
   Boundary pixels use reflection.

This produces a vector of `(intensity, noise_level)` observations.

#### Step 4: Curve Fitting (`OptimizeNoiseParameters`)

The 8-point noise LUT is fitted using Scaled Conjugate Gradient optimization
(Moller 1993). The loss function is:

```
loss = sum_observations [ asym * (interpolated_lut(intensity) - noise_level)^2 ]
     + kReg * num_observations * sum_i [ (w[i] - w[i+1])^2 ]
```

where:
- `asym = 1.0` if the fit undershoots (F(x) < noise_level), `1.1` if it overshoots
- The regularization term penalizes differences between adjacent LUT entries,
  encouraging a smooth curve
- `interpolated_lut` uses `IndexAndFrac` to linearly interpolate between the two
  nearest LUT entries based on the intensity (scaled by `(kNumNoisePoints-2) / 1.0 = 6`)

The optimizer is initialized with all LUT entries set to the mean noise level.
It runs up to 40 SCG iterations with precision 1e-8.

After optimization, LUT values are multiplied by `quality_coef * 1.4` and clamped
to [0, kNoiseLutMax]. If the final loss per observation exceeds kMaxError (1e-3),
the noise model is abandoned (all entries cleared).

#### Encoding (`EncodeNoise`)

Each of the 8 LUT entries is encoded as a 10-bit unsigned integer:
`round(value * kNoisePrecision)`, written as 10 bits each. Total: 80 bits for
the noise parameters.


## Patches

### Source Files

- `lib/jxl/dec_patch_dictionary.h` -- Decoder types: `PatchDictionary`, `PatchPosition`, `PatchReferencePosition`, `PatchBlending`, `PatchBlendMode`
- `lib/jxl/enc_patch_dictionary.h` -- Encoder: `PatchDictionaryEncoder`, `FindBestPatchDictionary`, `QuantizedPatch`
- `lib/jxl/enc_patch_dictionary.cc` -- Full encoder pipeline
- `lib/jxl/enc_dot_dictionary.h` -- Dot detection (alternative to text-like patches)

### Key Types

**`QuantizedPatch`** (in `enc_patch_dictionary.h`):
An encoder-side patch with both quantized (int8_t) and float pixel values.
Max size: 32x32 (`kMaxPatchSize`). Stores per-channel difference from the
background color. Supports comparison operators for deduplication sorting.

**`PatchInfo`** = `pair<QuantizedPatch, vector<pair<uint32_t,uint32_t>>>`:
A patch template paired with all its occurrence positions (x, y).

**`PatchColorspaceInfo`** (in `enc_patch_dictionary.cc`):
Channel-dependent quantization and distance weights, with separate values for
XYB vs non-XYB modes.

For XYB mode:
- `kChannelDequant` = {0.01615, 0.08875, 0.1922}
- `kChannelWeights` = {30.0, 3.0, 1.0}

For non-XYB:
- `kChannelDequant` = {20/255, 22/255, 20/255}
- `kChannelWeights` = {0.017*255, 0.02*255, 0.017*255}

**`PatchDictionary`** (in `dec_patch_dictionary.h`):
Decoder-side dictionary storing positions, reference positions, and blending
modes. Uses an interval tree on y-coordinates for efficient row-based lookup
during rendering.

**`PatchBlendMode`** enum: kNone, kReplace, kAdd, kMul, kBlendAbove/Below,
kAlphaWeightedAddAbove/Below. The encoder currently only produces kAdd patches
for text-like features and kNone for extra channels.

### Algorithm Details

#### When Patches Are Used

Patches are activated for "screenshot-like" images -- images that contain large
flat-color regions with small sharp foreground elements (text, icons). The
encoder detects these by looking for 4x4 pixel blocks where all pixels are
identical, surrounded by 8 neighbors (3x3 grid of 4x4 blocks) where at least
8 of 9 have the same color.

If no screenshot-like areas are found and patches are not explicitly forced,
the encoder falls back to **dot dictionary** detection instead, which is
enabled at speed <= Squirrel and butteraugli_distance >= 3.0
(`kMinButteraugliForDots`).

#### Patch Dictionary Construction (`FindTextLikePatches`)

1. **Screenshot detection**: Divide the image into a grid of 4x4 pixel blocks
   (`kPatchSide = 4`). Mark blocks where all 16 pixels have the same color
   AND at least 8 of 9 surrounding blocks share that color.

2. **Background flood-fill**: Starting from screenshot-like seed pixels, BFS
   with radius 1, propagating to "similar" neighbors (weighted color distance
   <= 0.8). Manhattan distance limit of 50 pixels from the source seed. The
   propagated color is the source seed's color, building a `background` image
   and `is_background` mask.

3. **Connected component extraction**: For each non-background pixel, find its
   connected component (8-connected, radius 1). Track bounding box. Skip CCs
   where:
   - No border pixel touches the background
   - Border pixels are not all similar to each other (threshold 0.03)
   - Bounding box exceeds 32x32 (`kMaxPatchSize`)

4. **Patch quantization**: For each valid CC, store the difference
   `opsin - background_color` quantized to int8 via channel-specific dequant
   factors. Skip patches where the maximum quantized absolute value < 2
   (`kMinPeak`).

5. **Deduplication**: Sort patches, merge duplicates. Remove patches that
   appear fewer than 2 times (`kMinPatchOccurrences`). Remove all patches if
   the largest patch is smaller than 20 pixels (`kMinMaxPatchSize`).

6. **Has-similar check**: For each CC, verify that at least one pixel in the
   vicinity (radius 2) of the bounding box has a color similar to the
   background reference color (threshold 0.03).

#### Reference Frame Construction

Patches are bin-packed into a reference frame using a first-fit algorithm:
- Start with dimensions max(largest_patch, sqrt(total_pixels))
- Grow by factor 1.05 + 1 pixel until all patches fit
- Mark occupied pixels to prevent overlap
- The reference frame is encoded as a separate modular-mode frame
  (reference ID 3, `kPatchFrameReferenceId`) with gradient predictor

#### Subtraction

`PatchDictionaryEncoder::SubtractFrom` removes patch contributions from the
opsin image before main encoding:
- kAdd mode: `opsin[pixel] -= reference[pixel]`
- kReplace mode: `opsin[pixel] = 0`
- kNone: no change

#### Encoding

Patches are entropy-coded using ANS with dedicated contexts for:
- Number of reference patches, reference frame ID
- Reference position (x0, y0), patch size
- Occurrence count and positions (delta-coded after first)
- Blending mode, alpha channel selection, clamping flag


## Splines

### Source Files

- `lib/jxl/splines.h` -- Shared types: `Spline`, `QuantizedSpline`, `Splines`, `SplineSegment`
- `lib/jxl/enc_splines.h` -- Encoder API: `EncodeSplines`, `FindSplines`
- `lib/jxl/enc_splines.cc` -- Encoder: `QuantizedSplineEncoder`, encoding logic

### Key Types

**`Spline`** (in `splines.h`):
An unquantized spline with:
- `control_points` -- vector of (x,y) float points defining the curve
- `color_dct[3]` -- three Dct32 arrays (32-entry DCT coefficients) for X, Y, B color along the spline
- `sigma_dct` -- Dct32 for the Gaussian splat width parameter along the spline

**`QuantizedSpline`** (in `splines.h`):
Integer-quantized version:
- `control_points_` -- vector of (int64_t, int64_t) pairs, **double-delta encoded**
- `color_dct_[3][32]` -- integer DCT coefficients for color
- `sigma_dct_[32]` -- integer DCT coefficients for sigma

**`SplineSegment`** (in `splines.h`):
A rendering unit for one row of a Gaussian splat:
- `center_x, center_y` -- position on the spline
- `maximum_distance` -- rendering radius
- `inv_sigma` -- 1/sigma for the Gaussian
- `sigma_over_4_times_intensity` -- precomputed rendering weight
- `color[3]` -- XYB color at this point

**`Splines`** (in `splines.h`):
Container for all splines in a frame:
- `quantization_adjustment_` -- global quantization precision modifier.
  If positive, multiply weights by (1 + adj/8); if negative, divide by (1 - adj/8)
- `splines_` -- vector of QuantizedSpline
- `starting_points_` -- one starting point per spline
- `segments_`, `segment_indices_`, `segment_y_start_` -- precomputed draw cache

Rendering: splines are drawn via normalized Gaussian splatting, with the
desired rendering distance of 1 pixel (`kDesiredRenderingDistance = 1.0`).

### Algorithm Details

#### Spline Fitting (Encoder)

**`FindSplines` is currently unimplemented** -- it returns an empty Splines object:
```cpp
Splines FindSplines(const Image3F& opsin) {
  // TODO(user): implement spline detection.
  return {};
}
```

The encoder never automatically detects splines from image content. Splines can
only be injected programmatically or through the API.

#### Encoding (`EncodeSplines`)

When splines are present, the encoding writes:
1. Number of splines minus 1 (kNumSplinesContext)
2. Starting points for all splines, delta-encoded after the first (kStartingPositionContext)
3. Quantization adjustment (kQuantizationAdjustmentContext), signed
4. For each spline:
   - Number of control points (kNumControlPointsContext)
   - Control point deltas as signed pairs (kControlPointsContext)
   - 3 x 32 color DCT coefficients as signed integers (kDCTContext)
   - 32 sigma DCT coefficients as signed integers (kDCTContext)

All values are entropy-coded using ANS with 6 contexts (`kNumSplineContexts`).

#### Rendering

Splines are rendered by evaluating the Catmull-Rom spline through the control
points, sampling at the desired rendering distance, and splatting each sample
as a Gaussian with the sigma derived from the sigma DCT. The color at each
sample comes from evaluating the color DCTs. CfL correlation (y_to_x, y_to_b)
is applied during dequantization to recover the full XYB color.


## Dependencies

- CfL depends on: AC strategy decisions, quantization matrices (`DequantMatrices`),
  the quantizer field, HWY SIMD, DCT transforms
- Noise depends on: `enc_optimize.h` (Scaled Conjugate Gradient method)
- Patches depend on: `enc_dot_dictionary`, `enc_frame` (for roundtrip encoding
  of the reference frame), `dec_frame` (for roundtrip decoding), ANS entropy coding
- Splines depend on: ANS entropy coding, CfL correlation (for dequantization)
- All features operate in the opsin (XYB) domain unless explicitly in non-XYB mode

## Open Questions

1. **CfL `kStrangeMultiplier = 128`**: The comment says "Experimentally values
   128-130 seem best -- I don't know why we need this multiplier." The purpose
   of this scaling between quantizer scale and the CfL optimization is unclear.
   It may relate to matching the dynamic range of quantized coefficients to the
   cost function's sensitivity.

2. **CfL towards-zero shrinkage**: The fixed threshold of 2.6 is noted as
   suboptimal. A variance-adaptive approach "would give about 1% more
   compression density" per the source comment but has not been implemented.

3. **CfL Newton epsilon = 100**: This is extremely large for a numerical
   derivative epsilon. It works because the derivatives are very noisy
   (many small absolute-value kinks in the cost), so a wide window averages
   out the noise. But it means the second derivative is a very coarse
   approximation.

4. **Noise SAD threshold**: The 0.15 upper limit and the mode-based thresholding
   have a TODO about bimodal and heavy-tailed histograms. Images with regular
   textures that produce a second peak are not handled.

5. **Spline detection**: `FindSplines` is unimplemented. The encoder cannot
   automatically discover splines. This limits the feature to synthetic or
   API-injected use cases.

6. **Patch extra channels**: The comment "patches must copy and use the real
   extra channels instead" indicates that patch blending for extra channels
   (alpha, depth, etc.) is not fully implemented -- placeholder zeros are used.

7. **Dot dictionary vs patches**: The comment notes "this doesn't work if both
   dots and patches are enabled." They are mutually exclusive: patches take
   priority, dots are only tried if no patches were found.

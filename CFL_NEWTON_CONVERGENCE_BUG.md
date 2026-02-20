# CfL Newton's Method Never Converges on Most Tiles

## Summary

`FindBestMultiplier` in `enc_chroma_from_luma.cc` uses Newton's method (`fast=false`) at effort >= 7 (speed_tier <= kSquirrel). The numerical derivative uses `eps=100`, which is far too large relative to typical coefficient magnitudes. Newton oscillates instead of converging on most tiles, producing arbitrary CfL values that hurt compression by 2-24%.

## Root Cause

The Newton loop (lines 159-168) computes an approximate second derivative via central difference with `eps=100`:

```cpp
float ddf = (dfpeps - dfmeps) / (2 * eps);  // eps = 100
```

The input coefficients are `dct_coeff * q * InvMatrix[i]`, where:
- Pass 1 (DCT8, q=1): typical `sum(a^2)` is 3-20
- Pass 2 (actual strategies): `sum(a^2)` ranges from ~50 to ~7M depending on distance and quant field

With `eps=100`, the function is evaluated at `x-100` and `x+100`. When the data scale (`sum_aa`) is small relative to `eps^2`, the second derivative estimate is dominated by the regularization term `distance_mul * x^2 * num` rather than the data term, causing Newton to overshoot and oscillate.

Measured on a 2048x1360 photo (07b9f93f from CLIC2025):

| Pass | Distance | Total tiles | Non-converged | Rate |
|------|----------|------------|---------------|------|
| 1 | d=1.0 | 1408 | 1408 | **100%** |
| 2 | d=1.0 | 1408 | 0 | 0% |
| 1 | d=5.0 | 1408 | 1408 | **100%** |
| 2 | d=5.0 | 1408 | 258 | **18%** |

Pass 1 **never converges** at any distance because `q=1` produces small magnitudes. Pass 2 converges at low distances (quant weighting amplifies magnitudes) but increasingly fails at high distances where `global_scale` shrinks.

## Observed Behavior

When Newton doesn't converge after 20 iterations, the code falls through and uses whatever `x` value the oscillation landed on. This is effectively random — it depends on iteration count parity.

Example tile at d=5.0:
- Newton oscillates: x = 5.55 → -8.38 → 5.55 → -8.38 ... (step ≈ 14, threshold = 3e-3)
- Final x depends on which side of the cycle we stop on
- `towards_zero` bias subtracts 2.6: `5.55 - 2.6 = 2.95 → round → 3`
- Least-squares optimum: `x_raw = -0.44` → after bias → **0**

The non-converged Newton produces `ytox=3` where the correct value is `ytox=0`.

## Compression Impact

Measured by comparing our encoder with LS fallback on non-convergence vs using the non-converged Newton value (matching libjxl behavior):

| Image | d=1.0 | d=3.0 | d=5.0 |
|-------|-------|-------|-------|
| 07b9f93f | +2.9% | +9.1% | +12.0% |
| 02809272 | +3.8% | +3.8% | +6.7% |
| 0369d229 | +4.0% | +6.0% | +8.3% |
| 0d154749 | +2.4% | +6.0% | +19.0% |
| 1b4ad095 | +5.4% | +18.6% | +23.8% |

Positive numbers = non-converged Newton produces larger files. The impact grows with distance because more tiles fail to converge.

## Suggested Fix

Track whether Newton converged. When it doesn't, fall back to the fast (least-squares) path:

```cpp
x = 0;
bool converged = false;
for (size_t i = 0; i < 20; i++) {
    // ... existing Newton iteration ...
    if (std::abs(step) < 3e-3) { converged = true; break; }
}
if (!converged) {
    // Fall back to least-squares
    auto ca = Zero(df);
    auto cb = Zero(df);
    const auto inv_color_factor = Set(df, 1.0f / kDefaultColorFactor);
    const auto base_v = Set(df, base);
    for (size_t i = 0; i < num; i += Lanes(df)) {
        const auto a = Mul(inv_color_factor, Load(df, values_m + i));
        const auto b = Sub(Mul(base_v, Load(df, values_m + i)), Load(df, values_s + i));
        ca = MulAdd(a, a, ca);
        cb = MulAdd(a, b, cb);
    }
    x = -GetLane(SumOfLanes(df, cb)) /
        (GetLane(SumOfLanes(df, ca)) + num * distance_mul * 0.5f);
}
```

Alternative: scale `eps` relative to the data magnitude, but this requires a pre-pass to compute `sum_aa` and adds complexity for no benefit over the LS fallback.

## How to Reproduce

Add `fprintf(stderr, ...)` after line 168 in `enc_chroma_from_luma.cc`:

```cpp
if (std::abs(step) >= 3e-3) {
    fprintf(stderr, "CFL newton_noconv num=%zu base=%.1f x=%.3f\n", num, base, x);
}
```

Encode any image at effort 7: `cjxl image.png out.jxl -d 5 -e 7`

Every tile will print at least once (pass 1), and many will print twice (pass 2) at d >= 3.0.

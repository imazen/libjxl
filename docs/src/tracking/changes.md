# Recent Changes (2026)

Changes merged to libjxl `main` since January 2026, organized by impact on encoder
behavior. Each entry links to the PR/commit and notes which encoder subsystems are
affected.

Last updated: 2026-02-20 against libjxl `main` at `03bafa6`.

## Encoder Bug Fixes

### Integer Overflow in AdjustQuantBlockAC

**PR [#4577](https://github.com/libjxl/libjxl/pull/4577)**, merged 2026-01-28.
Commit [`ae5cb19`](https://github.com/libjxl/libjxl/commit/ae5cb19).

Two bugs in [`enc_group.cc`](https://github.com/libjxl/libjxl/blob/main/lib/jxl/enc_group.cc)
`AdjustQuantBlockAC`:

1. **Activity computation overflow**: `hfNonZeros[i]` cast to `int32_t` before
   division. On HDR/extreme-range inputs, this overflows. Fix: compute
   `min_hf_non_zeros` as `float`, early-exit with `activity = 15` if below
   threshold, only then cast to `int32_t`.

2. **`orig_qp_limit` ordering**: Was computed before heuristics C and D could
   increase `*quant`. This meant the activity-reduction floor used a stale
   (lower) value, allowing activity reduction to undo the C/D increases. Fix:
   move `orig_qp_limit` computation after heuristics C and D.

**Before (vulnerable)**:
```cpp
int32_t activity = (int32_t(hfNonZeros[0]) + div/2) / div;  // overflow
int32_t orig_qp_limit = std::max(4, *quant / 2);            // too early
// ... heuristics C and D modify *quant ...
```

**After (fixed)**:
```cpp
float min_hf = std::min({hfNonZeros[0], hfNonZeros[1],
                         hfNonZeros[2], hfNonZeros[3]});
int32_t activity = 15;
if (min_hf < 15.0f * div) {
    activity = (static_cast<int32_t>(min_hf) + div / 2) / div;
}
// ... heuristics C and D ...
int32_t orig_qp_limit = std::max(4, *quant / 2);            // after C/D
```

**Affects**: VarDCT quantization, all effort levels. Quality impact on blocks with
both high-frequency corners (heuristic C) and low activity.

---

### Integer Overflow in FindTextLikePatches

**PR [#4568](https://github.com/libjxl/libjxl/pull/4568)**, merged 2026-01-27.
Commit [`2f10c05`](https://github.com/libjxl/libjxl/commit/2f10c05).

In [`enc_patch_dictionary.cc`](https://github.com/libjxl/libjxl/blob/main/lib/jxl/enc_patch_dictionary.cc),
`std::abs(val)` is undefined behavior when `val == INT_MIN`. Replaced with a
boolean check:

```cpp
// Before: max_value = std::max(max_value, std::abs(val));
// After:
is_small &= (val < kMinPeak) && (val > -kMinPeak);
```

**Affects**: Patch detection in VarDCT and lossless modes. Only triggers on
extreme HDR inputs where quantized values reach `INT_MIN`.

---

### Float-to-int Overflow in Patch Quantization

**PR [#4596](https://github.com/libjxl/libjxl/pull/4596)**, merged 2026-02-06.
Commit [`b6e9d19`](https://github.com/libjxl/libjxl/commit/b6e9d19).

Two fixes in [`enc_patch_dictionary.cc`](https://github.com/libjxl/libjxl/blob/main/lib/jxl/enc_patch_dictionary.cc):

1. `PatchColorspaceInfo::Quantize()` can produce values like `7.95e13` which
   overflow `int` when truncated. Fix: clamp to `[-32768, 32767]` before
   `std::trunc()`.

2. Patch values stored as `int8_t` were not range-checked. A quantized value
   of 500 would silently truncate. Fix: reject entire patch if any value
   doesn't fit in `int8_t` (same as the "too small" rejection path).

**Affects**: Patch dictionary encoding. Prevents distorted patch data on HDR
inputs.

---

### Integer Overflow in QuantizeWP

**PR [#4574](https://github.com/libjxl/libjxl/pull/4574)**, merged 2026-01-28.
Commit [`5a379aa`](https://github.com/libjxl/libjxl/commit/5a379aa).

`std::round(svalue)` in [`enc_modular.cc`](https://github.com/libjxl/libjxl/blob/main/lib/jxl/modular/encoding/enc_modular.cc)
`QuantizeWP()` overflows when `svalue` exceeds `int` range. Fix: bounds check
before rounding, set `has_outliers = true` and return error.

**Affects**: Modular VarDCT DC encoding with Weighted Predictor. Only triggers
on extreme float/HDR DC values.

---

## Quality Improvements

### Modular Effort Tradeoffs

**PR [#4236](https://github.com/libjxl/libjxl/pull/4236)**, merged 2026-02-13.
Commit [`b2edc77`](https://github.com/libjxl/libjxl/commit/b2edc77).

Three changes in [`enc_modular.cc`](https://github.com/libjxl/libjxl/blob/main/lib/jxl/modular/encoding/enc_modular.cc)
and [`enc_ma.cc`](https://github.com/libjxl/libjxl/blob/main/lib/jxl/modular/encoding/enc_ma.cc):

**1. Doubled `max_property_values` at all efforts:**

| Effort | Old | New |
|--------|-----|-----|
| e1-4 | 16 | 32 |
| e5 (Hare) | 24 | 48 |
| e6 (Wombat) | 32 | 64 |
| e7 (Squirrel) | 48 | 96 |
| e8 (Kitten) | 96 | 128 |
| e9-10 | 256 | 256 |

Enabled by a prior fix ([#4154](https://github.com/libjxl/libjxl/pull/4154))
that made `max_property_values` actually get enforced. More quantization buckets
improve tree density at minimal speed cost.

**2. Effort-modulated sampling fraction (`nb_repeats`):**

| Effort | Old | New |
|--------|-----|-----|
| e1-4 | 0.5 | 0.15 |
| e5 | 0.5 | 0.25 |
| e6 | 0.5 | 0.35 |
| e7 | 0.5 | 0.5 |
| e8 | 0.5 | 0.55 |
| e9-10 | 0.5 | 0.65 |

Formula: `nb_repeats = 0.5 * speed_factor` where `speed_factor` ranges from
0.3 (Lightning) to 1.3 (Glacier).

**3. Simplified WP split categorization in `FindBestSplit`:**

Replaced `adds_wp` tracking (was WP already used by an ancestor?) with simpler
`uses_wp` boolean. Fixes edge case where a parent already using WP caused
a child WP-split to be mis-categorized in the priority system. Also removed
`used_properties` bitmask from `NodeInfo`.

The `fast_decode_multiplier` (1.01 at effort < 7, 1.0 at effort >= 7) prefers
non-WP splits when cost is within 1%, favoring decode speed.

**Affects**: Modular lossless compression at all effort levels. ~0.1% density
improvement from more property buckets, speed improvement at lower efforts from
reduced sampling.

---

## Decoder Fixes (Non-Encoder)

These don't affect the bitstream format but are worth noting for context:

- **CMS grayscale crash** (PR [#4579](https://github.com/libjxl/libjxl/pull/4579),
  Feb 4): Buffer interleaving bug in `stage_cms.cc` when source or destination
  is grayscale.

- **Dithering LUT fix** (commit `a5ee1a6`, Jan 28): Blue noise dither pattern
  in `stage_write.cc` now uses per-channel offsets to avoid correlated dithering.

- **Primary color representation** (commit `4867d7b`, Feb 10): Fixed logic in
  `dec/jxl.cc` for determining ICC vs enum color encoding primacy.

- **Box content decoder underflow** (commit `39ed990`, Jan 29): Unsigned
  underflow in container box pointer adjustment.

## Security Backports

All encoder overflow fixes above were backported to maintenance branches v0.7.3
through v0.11.2 (PRs #4601-4616). Google Project Zero involvement on the
`PackedImage` memory size overflow (PR #4589). These are real attack surface
issues, not theoretical — fuzz-discovered on production inputs.

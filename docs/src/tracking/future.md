# Future Directions

Open PRs, feature requests, and architectural directions being explored in
libjxl. These may change encoder behavior when merged.

Last updated: 2026-02-20.

## Open PRs — Encoder Quality

### VarDCT Block Size Selection Tuning

**PR [#4506](https://github.com/libjxl/libjxl/pull/4506)**, open.
Author: jonnyawsom3.

Tunes the cost model constants in [`enc_ac_strategy.cc`](https://github.com/libjxl/libjxl/blob/main/lib/jxl/enc_ac_strategy.cc).
This is a significant calibration change that would shift RD tradeoffs for all
AC strategy decisions.

**8x8-class strategy multipliers:**

| Strategy | Old | New |
|----------|-----|-----|
| DCT8 | 0.80 | 0.83 |
| DCT4x4 | 1.08 | 0.78 |
| DCT2x2 | 0.95 | 0.90 |
| DCT4x8/DCT8x4 | 0.859 | 0.80 |
| IDENTITY | 1.04 | 0.93 |
| AFV0-3 | 0.818 | 0.78 |

**entropy_mul for large transforms:**

| Strategy | Old | New |
|----------|-----|-----|
| DCT16x8/DCT8x16 | 1.21 | 1.25 |
| DCT16x16 | 1.34 | 1.35 |
| DCT16x32/DCT32x16 | 1.49 | 1.60 |
| DCT32x32 | 1.48 | 1.75 |
| DCT64x32/DCT32x64 | 2.25 | 2.20 |
| DCT64x64 | 2.25 | 2.50 |

**Distance-dependent constants:**

| Constant | Old | New |
|----------|-----|-----|
| `info_loss_multiplier` | 1.2 | 1.3 |
| `zeros_mul` | 9.309 | 9.31 |
| `cost_delta` | 10.833 | 10.8 |
| `kBias` | 0.137 | 0.14 |

Net effect: larger transforms get penalized more (especially DCT32x32:
1.48 to 1.75), small transforms become relatively cheaper (DCT4x4: 1.08 to
0.78). The distance-dependent scaling shifts slightly.

Note: The PR has `JXL_DEBUG_AC_STRATEGY 1` left on — likely needs cleanup
before merge.

---

### Better JPEG Density via CfL Averaging

**PR [#4480](https://github.com/libjxl/libjxl/pull/4480)**, open.
Author: Melirius.

Changes `FindIndexOfSumMaximum` to `FindAvgIndexOfSumMaximum` in
[`enc_frame.cc`](https://github.com/libjxl/libjxl/blob/main/lib/jxl/enc_frame.cc).
Instead of taking the first bin index at the maximum cumulative sum, takes the
average of the first and last bin indices: `(first + last + 1) >> 1`.

Only affects JPEG recompression CfL computation. ~0.003% density improvement.

---

### Allow More Predictors for Patch Reference Frames

**PR [#4533](https://github.com/libjxl/libjxl/pull/4533)**, draft, stalled.
Author: jonnyawsom3.

Currently patch reference frames are encoded with Gradient predictor hardcoded.
This PR enables all predictors except Weighted. Early results showed a speed
regression, so development paused.

Tree learning on patch reference frames could improve density for screenshot
content (text glyphs, UI elements). The reference frame is small (at most
256x256) so tree learning overhead is minimal.

---

## Open PRs — Architecture

### Buffered Encoding (Non-Streaming)

**PR [#4580](https://github.com/libjxl/libjxl/pull/4580)**, open.
Author: veluca93.

Major refactoring of [`enc_frame.cc`](https://github.com/libjxl/libjxl/blob/main/lib/jxl/enc_frame.cc)
to support buffered encoding — encode the entire frame in memory before writing
output. This decouples encoding order from output order, enabling better
progressive decoding without streaming constraints.

Follow-up **PR [#4611](https://github.com/libjxl/libjxl/pull/4611)** exposes
`JXL_ENC_FRAME_SETTING_BUFFERING` in the API:
- Level 0: streaming (current default)
- Level 1: buffer large sections
- Level 2: buffer all multi-group images

Claims 8-10x encode speed improvement for 1080p images at level 2.

This confirms that full buffering (encoding everything in memory before writing)
is the preferred approach for quality and speed. The streaming path remains for
memory-constrained environments.

---

### Reference Frame 3 Relaxation

**PR [#4512](https://github.com/libjxl/libjxl/pull/4512)**, open.

Relaxes the restriction on reference frame slot 3, which is currently reserved
for the encoder's internal use (patches). The PR allows user-specified reference
frame 3 when all frames have patches explicitly disabled.

Relevant for multi-frame encoding (animation, layered compositing) where the
4-slot limit is constraining.

---

## Open PRs — Decoder

### Change Default Color Decoding to sRGB

**PR [#4516](https://github.com/libjxl/libjxl/pull/4516)**, open.

Changes the default output transfer curve from Linear to sRGB. Linear output
causes banding when decoded to 8-bit integer buffers. The sRGB transfer function
preserves more perceptual precision in the 8-bit quantization.

This is a decode-side API default change — does not affect the bitstream.

---

### Lenient Colorspace Check for kReplace Blending

**PR [#4423](https://github.com/libjxl/libjxl/pull/4423)**, open.
Label: `spec`.

Makes the decoder accept `xyb_encoded=true && want_icc=true` when the blend
mode is `kReplace`, since no actual color-space-dependent blending occurs.
Currently the spec prohibits this combination.

See [Known Issues — XYB + ICC + Frame Blending](issues.md#xyb--icc--frame-blending-is-spec-invalid).

---

## Feature Requests

### Optimize FindTextLikePatches

**Issue [#4334](https://github.com/libjxl/libjxl/issues/4334)**, open.

Performance optimization for patch detection. The `is_same` pixel comparison
loop is the hot path. Discussion includes:
- Strip-level comparison (compare entire rows at once)
- SIMD vectorization of the L1 distance computation
- Early exit on first mismatch per strip

For large screenshot images (4K+), patch detection can dominate encode time.

---

### JxlEncoderParseOptions

**Issue [#4532](https://github.com/libjxl/libjxl/issues/4532)**, open.

Feature request for a function to parse cjxl-style option strings directly,
enabling embedding cjxl's CLI syntax in other tools. API-level feature for the
C library, no encoder algorithm impact.

---

## Version History

### Maintenance Releases (2026)

Security fixes backported to all supported branches:

| Branch | Release | Key Fixes |
|--------|---------|-----------|
| v0.7.x | v0.7.3 | Overflow fixes, CMS crash |
| v0.8.x | v0.8.5 | Overflow fixes, CMS crash |
| v0.9.x | v0.9.5 | Overflow fixes, CMS crash |
| v0.10.x | v0.10.5 | Overflow fixes, CMS crash |
| v0.11.x | v0.11.2 | Overflow fixes, CMS crash |

PRs #4601-4616 cover the backports. Google Project Zero involvement on the
`PackedImage` memory size overflow (PR #4589).

The current development head is post-v0.12.0, with no v0.13 release yet.

# Known Issues

Open bugs and spec-level issues in libjxl that affect encoder behavior or
bitstream correctness.

Last updated: 2026-02-20.

## Encoder Bugs

### MA Tree Node Limit Exceeded

**Issue [#4600](https://github.com/libjxl/libjxl/issues/4600)**, open.

`cjxl -d 0 -e 10 -E 11 -I 100 -g 3` can produce files where the MA tree
exceeds the spec maximum of 61024 nodes, making them un-decodable.

The spec limit is:
```
max_tree_size = min(61024, 1 << (2 * max(ceil(log2(xsize)), ceil(log2(ysize)))))
```

Triggered by stereogram images with high `max_property_values` and extreme
effort settings. Tree learning must enforce this cap during construction.

**Affected code**: [`enc_ma.cc`](https://github.com/libjxl/libjxl/blob/main/lib/jxl/modular/encoding/enc_ma.cc)
`FindBestSplit`

---

### Modular Residual Overflow

**Issue [#4620](https://github.com/libjxl/libjxl/issues/4620)**, open.

`value - predicted` overflows `int32_t` when `predicted` is `INT_MIN`:
`0 - (-2147483648)` wraps to `-2147483648`. Occurs in
[`enc_encoding.cc:306`](https://github.com/libjxl/libjxl/blob/main/lib/jxl/modular/encoding/enc_encoding.cc#L306)
during modular channel encoding.

Related: Issue [#4593](https://github.com/libjxl/libjxl/issues/4593) — similar
overflow in `ComputeEncodingData` (`enc_modular.cc:793`) during patch dictionary
roundtrip with extreme float input.

Both are fuzz-discovered. Fix needs `int64_t` intermediates or clamped
subtraction in the residual computation path.

**Affected code**: [`enc_encoding.cc`](https://github.com/libjxl/libjxl/blob/main/lib/jxl/modular/encoding/enc_encoding.cc)
`EncodeModularChannelMAANS`

---

### VarDCT Blocking with `uses_original_profile=true`

**Issue [#4552](https://github.com/libjxl/libjxl/issues/4552)**, open.

VarDCT encoding with `uses_original_profile=true` (non-XYB, RGB-domain lossy)
produces severe blocking artifacts at effort 7+. Reproducible with:
```
cjxl --disable_perceptual_optimizations -d 1 -e 7 input.png output.jxl
```

Only VarDCT is affected; modular mode is fine. The adaptive quantization and
perceptual masking are tuned for XYB and don't work correctly in RGB domain.
Not a regression — has been broken for a long time.

**Affected code**: [`enc_adaptive_quantization.cc`](https://github.com/libjxl/libjxl/blob/main/lib/jxl/enc_adaptive_quantization.cc),
XYB-specific masking applied to non-XYB data.

---

### Group Palette Compression Regression

**Issue [#4387](https://github.com/libjxl/libjxl/issues/4387)**, open.

Local (per-group) channel palette at the default `-Y 80` threshold hurts lossless
compression in many cases since v0.11. Disabling it (`-Y 0`) produces smaller
files. The regression is caused by interactions between chunked encoding (which
breaks global palette) and group palette overhead.

Related: Issue [#4379](https://github.com/libjxl/libjxl/issues/4379) — broader
lossless density regression between v0.8 and v0.11, bisected to commits
affecting palette and chunked encoding.

**Affected code**: [`enc_modular.cc`](https://github.com/libjxl/libjxl/blob/main/lib/jxl/modular/encoding/enc_modular.cc)
palette transform threshold

---

### Lossless + Gaborish Doesn't Roundtrip

**Issue [#4482](https://github.com/libjxl/libjxl/issues/4482)**, open.

Combining `-d 0` with `--gaborish=1` produces non-lossless output. This is
expected behavior — gaborish inverse is an approximate pre-filter, not an exact
inverse. The decoder's forward gaborish doesn't perfectly cancel the encoder's
inverse. libjxl disables gaborish at `distance=0`, but the API allows forcing
it on.

Not a bug per se, but a footgun. Encoders should reject or warn on
`distance=0 + gaborish=on`.

---

### RCT Selection Broken for Progressive Lossless

**Issue [#4388](https://github.com/libjxl/libjxl/issues/4388)**, open.

RCT selection was broken for progressive lossless images, producing ~20% larger
files. Was forced to YCoCg as a workaround, but not all images benefit from
YCoCg. The root cause is that progressive mode's coefficient splitting
interferes with the RCT trial encode.

**Affected code**: [`enc_modular.cc`](https://github.com/libjxl/libjxl/blob/main/lib/jxl/modular/encoding/enc_modular.cc)
RCT selection path for progressive mode

---

## Spec-Level Issues

### JPEG Reconstruction `reset_points` Limit

**Issue [#4310](https://github.com/libjxl/libjxl/issues/4310)**, open.

The `jbrd` box's `reset_points` field uses `U32(Val(0), BitsOffset(2,1),
BitsOffset(4,4), BitsOffset(16,20))` which caps at 65555. Large JPEGs (e.g.,
5616x3744) exceed this with their restart marker indices.

Discussion includes:
- Extending the spec with a wider encoding
- Adding a "non-bitexact" reconstruction mode
- Handling Ultra HDR gain map JXLs

The spec may be amended in a future revision.

---

### XYB + ICC + Frame Blending Is Spec-Invalid

**Issue [#4419](https://github.com/libjxl/libjxl/issues/4419)**, open.

The combination `xyb_encoded=true && want_icc=true && save_before_ct=false`
is rejected by libjxl's decoder but accepted by jxl-oxide and jxlatte.
Triggered by hydrium-encoded multi-frame VarDCT with ICC profiles.

PR [#4423](https://github.com/libjxl/libjxl/pull/4423) proposes relaxing
the decoder to allow this when the blend mode is `kReplace` (since no actual
color-space-dependent blending occurs). The spec currently prohibits it.

**Implication for encoders**: When emitting multi-frame VarDCT with ICC profiles,
use `save_before_ct=true` or ensure `kReplace` blend mode until the spec is
clarified.

---

### Butteraugli MaltaUnit Pattern Duplication

**Issue [#4623](https://github.com/libjxl/libjxl/issues/4623)**, closed.

MaltaUnit patterns 13-16 in libjxl are duplicates of patterns 5-8. The
standalone `google/butteraugli` has different S-curve patterns. The codebases
have diverged. libjxl's version is authoritative for JPEG XL encoder tuning.

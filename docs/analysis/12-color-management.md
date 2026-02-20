# Color Management & Transfer Functions

Analysis of libjxl's color management system: transfer functions, CMS interface,
tone mapping, and color encoding signaling.

## Source Files

| File | Purpose |
|------|---------|
| `lib/jxl/cms/transfer_functions.h` | Scalar base classes for TF_HLG_Base, TF_PQ_Base |
| `lib/jxl/cms/transfer_functions-inl.h` | SIMD-accelerated transfer functions (TF_HLG, TF_PQ, TF_SRGB, TF_709, FastLinearToSRGB) |
| `lib/jxl/cms/tone_mapping.h` | Scalar base classes for Rec2408ToneMapperBase, HlgOOTF_Base, GamutMapScalar |
| `lib/jxl/cms/tone_mapping-inl.h` | SIMD-accelerated tone mapping (Rec2408ToneMapper, HlgOOTF, GamutMap) |
| `lib/jxl/cms/jxl_cms.cc` | Main CMS implementation -- JxlCmsInit, DoColorSpaceTransform, ICC profile parsing |
| `lib/jxl/cms/jxl_cms_internal.h` | ICC profile generation, tone map pixel, Bradford matrices, ExtraTF enum |
| `lib/jxl/cms/color_encoding_cms.h` | Core color encoding types: ColorSpace, WhitePoint, Primaries, TransferFunction, etc. |
| `lib/jxl/cms/opsin_params.h` | XYB color space constants (opsin absorbance matrix, bias, scale) |
| `lib/jxl/color_encoding_internal.h` | Internal ColorEncoding struct wrapping cms::ColorEncoding, ColorSpaceTransform |
| `lib/include/jxl/cms_interface.h` | Public C API for the pluggable CMS callback system |
| `lib/extras/tone_mapping.h` | High-level ToneMapTo / GamutMap functions for CodecInOut |


## Key Types

### ColorSpace (enum, `color_encoding_cms.h:39`)
```
kRGB      = 0   Trichromatic (also used for CMYK with kBlack extra channel)
kGray     = 1   Single-channel
kXYB      = 2   Fixed primaries, implies D65 white point
kUnknown  = 3   Non-RGB/gray sensor data
```

### WhitePoint (enum, `color_encoding_cms.h:58`)
Values from CICP ColourPrimaries:
```
kD65    = 1    sRGB/BT.709/Display P3/BT.2020   xy = (0.3127, 0.3290)
kCustom = 2    Actual values encoded in separate fields
kE      = 10   Equal-energy illuminant (XYZ)     xy = (1/3, 1/3)
kDCI    = 11   DCI-P3                            xy = (0.314, 0.351)
```

### Primaries (enum, `color_encoding_cms.h:67`)
Values from CICP ColourPrimaries:
```
kSRGB   = 1    Same as BT.709
                 R = (0.639998686, 0.330010138)
                 G = (0.300003784, 0.600003357)
                 B = (0.150002046, 0.059997204)
kCustom = 2    Actual values in separate fields
k2100   = 9    BT.2020/2100
                 R = (0.708, 0.292)
                 G = (0.170, 0.797)
                 B = (0.131, 0.046)
kP3     = 11   DCI-P3
                 R = (0.680, 0.320)
                 G = (0.265, 0.690)
                 B = (0.150, 0.060)
```
Note: sRGB primaries use quantized values from ICC 15-bit fixed-point, not the
standard spec values (0.64, 0.33, ...).

### TransferFunction (enum, `color_encoding_cms.h:76`)
Values from CICP TransferCharacteristics:
```
k709     = 1    BT.709
kUnknown = 2    Cannot be represented by known enum
kLinear  = 8    Identity (gamma 1.0)
kSRGB    = 13   sRGB
kPQ      = 16   Perceptual Quantizer (BT.2100)
kDCI     = 17   DCI (gamma 2.6)
kHLG     = 18   Hybrid Log-Gamma (BT.2100)
```

### RenderingIntent (enum, `color_encoding_cms.h:87`)
Values match ICC encoding:
```
kPerceptual  = 0   Good for photos; requires LUT profile
kRelative    = 1   Good for logos
kSaturation  = 2   Fully saturated CG
kAbsolute    = 3   Leaves white point unchanged; good for proofing
```

### CustomTransferFunction (`color_encoding_cms.h:201`)
```cpp
struct CustomTransferFunction {
    static constexpr uint32_t kMaxGamma = 8192;       // max 1/(1/8192)
    static constexpr uint32_t kGammaMul = 10000000;    // fixed-point multiplier

    bool have_gamma = false;
    uint32_t gamma = 0;                                // stored as gamma * kGammaMul
    TransferFunction transfer_function = TransferFunction::kSRGB;
};
```
`SetGamma(g)` auto-detects: gamma ~1.0 -> kLinear, gamma ~1/2.6 -> kDCI.
Valid gamma range: `(1/8192, 1.0]`.

### Customxy (`color_encoding_cms.h:112`)
Serializable chromaticity coordinates:
```
kMul = 1000000        Fixed-point multiplier for CIExy values
kRoughLimit = 4.0     Max absolute value for x or y
kMin = -0x200000      -2097152 in fixed point
kMax = 0x1FFFFF       2097151 in fixed point
```

### ColorEncoding (`color_encoding_cms.h:323`)
Core internal color encoding struct:
```cpp
struct ColorEncoding {
    WhitePoint white_point = WhitePoint::kD65;
    Primaries primaries = Primaries::kSRGB;
    RenderingIntent rendering_intent = RenderingIntent::kRelative;
    bool have_fields = true;
    IccBytes icc;
    ColorSpace color_space = ColorSpace::kRGB;
    bool cmyk = false;
    CustomTransferFunction tf;
    Customxy white, red, green, blue;   // custom xy fields
};
```

### JxlCms (`jxl_cms.cc:54`)
Internal CMS state, allocated by JxlCmsInit:
```cpp
struct JxlCms {
    // skcms or LCMS backend profiles/transforms
    bool apply_hlg_ootf;
    size_t hlg_ootf_num_channels;
    std::array<float, 3> hlg_ootf_luminances;  // Y of primaries
    size_t channels_src, channels_dst;
    std::vector<float> src_storage, dst_storage;
    std::vector<float*> buf_src, buf_dst;       // per-thread interleaved buffers
    float intensity_target;
    bool skip_lcms = false;
    ExtraTF preprocess = ExtraTF::kNone;        // applied before CMS
    ExtraTF postprocess = ExtraTF::kNone;       // applied after CMS
};
```

### ExtraTF (enum, `jxl_cms_internal.h:36`)
Identifies transfer functions that libjxl handles directly (outside of
skcms/LCMS), using its own SIMD implementations:
```
kNone   No extra processing
kPQ     Perceptual Quantizer
kHLG    Hybrid Log-Gamma
kSRGB   sRGB
```

### JxlColorProfile (`cms_interface.h:47`)
C API struct passed to the CMS init function:
```c
typedef struct {
    struct { const uint8_t* data; size_t size; } icc;
    JxlColorEncoding color_encoding;
    size_t num_channels;
} JxlColorProfile;
```

### JxlCmsInterface (`cms_interface.h:227`)
Pluggable CMS callback table:
```c
typedef struct {
    void* set_fields_data;
    jpegxl_cms_set_fields_from_icc_func set_fields_from_icc;
    void* init_data;
    jpegxl_cms_init_func init;
    jpegxl_cms_get_buffer_func get_src_buf;
    jpegxl_cms_get_buffer_func get_dst_buf;
    jpegxl_cms_run_func run;
    jpegxl_cms_destroy_func destroy;
} JxlCmsInterface;
```

### ColorSpaceTransform (`color_encoding_internal.h:306`)
RAII wrapper for the CMS interface lifecycle:
```cpp
class ColorSpaceTransform {
    Status Init(const ColorEncoding& c_src, const ColorEncoding& c_dst,
                float intensity_target, size_t xsize, size_t num_threads);
    float* BufSrc(size_t thread) const;
    float* BufDst(size_t thread) const;
    Status Run(size_t thread, const float* buf_src, float* buf_dst, size_t xsize);
};
```


## Key Functions

### CMS Interface Implementation

**`JxlGetDefaultCms()`** (`jxl_cms.cc:1352`)
Returns the default `JxlCmsInterface` singleton. The interface struct is:
```
set_fields_data    = nullptr
set_fields_from_icc = JxlCmsSetFieldsFromICC
init_data          = pointer to the interface itself
init               = JxlCmsInit
get_src_buf        = JxlCmsGetSrcBuf
get_dst_buf        = JxlCmsGetDstBuf
run                = DoColorSpaceTransform
destroy            = JxlCmsDestroy
```

**`JxlCmsSetFieldsFromICC(void* user_data, const uint8_t* icc_data, size_t icc_size, JxlColorEncoding* c, JXL_BOOL* cmyk)`** (`jxl_cms.cc:954`)
Parses an ICC profile and populates a `JxlColorEncoding`. Steps:
1. Parse with skcms/LCMS.
2. Extract rendering intent from bytes 60-63 (big-endian, tolerates little-endian).
3. Check for CICP tag -- if present and recognized, use `ApplyCICP()` to set fields directly.
4. Determine color space (RGB/Gray/CMYK/Unknown).
5. Extract unadapted white point (undo chromatic adaptation if CHAD tag present).
6. Identify primaries (transform unit RGB vectors to XYZ, undo chromatic adaptation, convert to xy).
7. Detect transfer function by trying gamma match first, then iterating all known `TransferFunction` values and comparing ICC profiles via `IsApproximatelyEqual()` (skcms) or `ProfileEquivalentToICC()` (LCMS).

**`JxlCmsInit(void* init_data, size_t num_threads, size_t xsize, const JxlColorProfile* input, const JxlColorProfile* output, float intensity_target)`** (`jxl_cms.cc:1105`)
Creates a color transform. Key decision logic:
1. Parse both ICC profiles.
2. If `c_src.SameColorEncoding(c_dst)` -> set `skip_lcms = true`.
3. Determine if HLG OOTF is needed: `apply_hlg_ootf = c_src.tf.IsHLG() != c_dst.tf.IsHLG()`.
4. **ExtraTF optimization**: If source is PQ, HLG, or (sRGB with same-color-space linear dest), replace the source profile with a linear version and set `preprocess` to handle the TF via libjxl's SIMD path instead of skcms/LCMS. Same logic for destination with `postprocess`.
5. After ExtraTF substitution, re-check if intermediary profiles match -> `skip_lcms = true`.
6. Create skcms/LCMS transform from the (possibly linearized) profiles.
7. Allocate per-thread interleaved float buffers (128-byte aligned).

**`DoColorSpaceTransform(void* cms_data, size_t thread, const float* buf_src, float* buf_dst, size_t xsize)`** (`jxl_cms.cc:204`)
Pipeline for each row:
1. **Preprocess** (`BeforeTransform`): Apply inverse TF via SIMD (PQ->linear, HLG->linear, sRGB->linear). For HLG, also applies forward HLG OOTF.
2. **Channel expansion/conversion**: Expand grayscale 1ch to 3ch (skcms) or invert CMYK values for LCMS (`100 - 100*x`).
3. **CMS transform**: `skcms_Transform()` or `cmsDoTransform()`, or memcpy if `skip_lcms`.
4. **Channel contraction**: Contract 3ch back to 1ch for grayscale dest (skcms).
5. **Postprocess** (`AfterTransform`): Apply forward TF via SIMD (linear->PQ, linear->HLG, linear->sRGB). For HLG, applies inverse OOTF first.

**`ApplyHlgOotf(JxlCms* t, float* buf, size_t xsize, bool forward)`** (`jxl_cms.cc:857`)
Applies the HLG OOTF in-place. Skipped if `intensity_target` is in `[295, 305]` (gamma ~1.0).
```
gamma = 1.2 * pow(1.111, log2(intensity_target / 1000))
if (!forward) gamma = 1/gamma
```
For 3-channel: `ratio = pow(luminance, gamma - 1)`, each channel multiplied by `ratio`.
If forward and gamma < 1, out-of-gamut highlights are normalized by dividing by max component.

**`GetPrimariesLuminances(const ColorEncoding& encoding, float luminances[3])`** (`jxl_cms.cc:810`)
Computes the Y-row of the RGB-to-XYZ matrix for the given primaries and white point.
Uses matrix inversion of the primaries chromaticity matrix.

### Transfer Function Application

**`BeforeTransform(JxlCms* t, const float* buf_src, float* xform_src, size_t buf_size)`** (`jxl_cms.cc:98`)
Undoes gamma compression. Dispatches on `t->preprocess`:
- `kPQ`: `TF_PQ(intensity_target).DisplayFromEncoded()`
- `kHLG`: `TF_HLG_Base::DisplayFromEncoded()` then optional OOTF
- `kSRGB`: `TF_SRGB().DisplayFromEncoded()`

**`AfterTransform(JxlCms* t, float* buf_dst, size_t buf_size)`** (`jxl_cms.cc:155`)
Applies gamma compression. Dispatches on `t->postprocess`:
- `kPQ`: `TF_PQ(intensity_target).EncodedFromDisplay()`
- `kHLG`: Optional inverse OOTF then `TF_HLG_Base::EncodedFromDisplay()`
- `kSRGB`: `TF_SRGB().EncodedFromDisplay()`

### ICC Profile Generation

**`MaybeCreateProfile(const JxlColorEncoding& c, std::vector<uint8_t>* icc)`** (`jxl_cms_internal.h:1123`)
Creates an ICC profile from structured color encoding fields. Generates:
- ICC v4.4 header with `"jxl "` CMM tag.
- CICP tag for known primaries/TF combinations.
- chromatic adaptation (chad) tag using Bradford transform to D50.
- RGB colorant tags (rXYZ, gXYZ, bXYZ) from primaries-to-XYZD50 matrix.
- Transfer function curves:
  - sRGB: `para` type 3 with `{2.4, 1/1.055, 0.055/1.055, 1/12.92, 0.04045}`
  - BT.709: `para` type 3 with `{1/0.45, 1/1.099, 0.099/1.099, 1/4.5, 0.081}`
  - Linear: `para` type 3 with `{1.0, 1.0, 0.0, 1.0, 0.0}`
  - DCI: `para` type 0 with `{2.6}`
  - Gamma: `para` type 0 with `{1/gamma}`
  - PQ: `curv` 64-entry table from `CreateTableCurve<64, ExtraTF::kPQ>()`
  - HLG: `curv` 64-entry table from `CreateTableCurve<64, ExtraTF::kHLG>()`
- For XYB: `mAB` (A2B0) tag with 2x2x2 CLUT + parametric M curves + matrix.
- For HDR tone-mappable (PQ/HLG with known primaries): `mft1` (A2B0) 9x9x9 3D LUT to CIELAB PCS.
- Profile ID via MD5 checksum.

**`ToneMapPixel(const JxlColorEncoding& c, const float in[3], uint8_t pcslab_out[3])`** (`jxl_cms_internal.h:128`)
Per-pixel tone mapping for 3D ICC LUT generation. Steps:
1. Convert encoded to linear via PQ (at 10000 nits) or HLG EOTF.
2. PQ: apply Rec.2408 tone map from `[0, 10000]` to `[0, 250]` nits.
   HLG: apply HlgOOTF from 300 nit source to 80 nit target.
3. Apply gamut mapping with `preserve_saturation = 0.3`.
4. Convert to XYZ D50 via chromatic adaptation.
5. Convert XYZ to CIELAB and encode as uint8.


## Constants

### Default Intensity Target
```
kDefaultIntensityTarget = 255     (lib/jxl/base/common.h:61)
```
The luminance level (cd/m^2) at which linear 1.0 is displayed for non-PQ, non-HLG content.

### PQ Constants (`transfer_functions.h:121`)
From BT.2100-2 / SMPTE ST 2084:
```
kM1 = 2610.0 / 16384             = 0.1593017578125
kM2 = (2523.0 / 4096) * 128      = 78.84375
kC1 = 3424.0 / 4096              = 0.8359375
kC2 = (2413.0 / 4096) * 32       = 18.8515625
kC3 = (2392.0 / 4096) * 32       = 18.6875
```

### HLG Constants (`transfer_functions.h:82`)
From BT.2100-2:
```
kA     = 0.17883277
kRA    = 1.0 / kA                 = 5.591816309728916...
kB     = 1 - 4 * kA               = 0.28466892
kC     = 0.5599107295
kInv12 = 1.0 / 12.0               = 0.08333...
```

### HLG SIMD-specific derived constants (`transfer_functions-inl.h:86`)
```
k05    = 0.5
k3     = 3.0
kInv3  = 1.0 / 3.0
kHiAdd = kB * kInv12              = kB / 12
kHiMul = 0.003639807079052639     = exp(-kC * kRA) * kInv12
kHiPow = 8.067285659607931        = kRA * log2(e)
kInvLog2e = 1/log2(e)             (used implicitly in EncodedFromDisplay)
```

### sRGB Thresholds (`transfer_functions-inl.h:271`)
```
kThreshSRGBToLinear = 0.04045     Encoded -> Linear threshold (on encoded value)
kThreshLinearToSRGB = 0.0031308   Linear -> Encoded threshold (on linear value)
kLowDiv            = 12.92        Linear segment slope
kLowDivInv         = 1.0 / 12.92  Inverse for decoding
```

### BT.709 Constants (`transfer_functions-inl.h:120`)
```
kThresh    = 0.018       Linear-to-encoded threshold (on linear value)
kMulLow    = 4.5         Linear segment slope
kMulHi     = 1.099       Power segment scale
kPowHi     = 0.45        Power segment exponent
kSub       = -0.099      Power segment offset

kInvThresh = 0.081       Encoded-to-linear threshold (on encoded value)
kInvMulLow = 1 / 4.5
kInvMulHi  = 1 / 1.099
kInvPowHi  = 1 / 0.45
kInvAdd    = 0.099 / 1.099
```

### Bradford Chromatic Adaptation (`jxl_cms_internal.h:72`)
```
kBradford = {{0.8951,  0.2664, -0.1614},
             {-0.7502, 1.7135,  0.0367},
             {0.0389, -0.0685,  1.0296}}

kBradfordInv = {{0.9869929, -0.1470543, 0.1599627},
                {0.4323053,  0.5183603, 0.0492912},
                {-0.0085287, 0.0400428, 0.9684867}}
```

### D50 White Point
ICC PCS illuminant (quantized):
```
D50 XYZ = (0.96420288, 1.0, 0.82490540)    (jxl_cms.cc:337, skcms path)
D50 XYZ = (0.96422,    1.0, 0.82521)        (jxl_cms_internal.h:90, Bradford adaptation target)
D50 xy  via ICC header = (0.9642/65536, 1.0/65536, 0.8249/65536) fixed-point
```

### XYB / Opsin Absorbance (`opsin_params.h`)

**Opsin absorbance matrix:**
```
M = {{0.30,          0.622,       0.078},
     {0.23,          0.692,       0.078},
     {0.24342268..., 0.20476744..., 0.55180987...}}
```
Where `kM01 = 1 - 0.078 - 0.30 = 0.622`, `kM11 = 1 - 0.078 - 0.23 = 0.692`,
`kM22 = 1 - 0.24342268... - 0.20476744... = 0.55180987...`.

**Opsin absorbance bias (all three channels identical):**
```
kOpsinAbsorbanceBias = 0.0037930732552754493
```

**Inverse opsin absorbance matrix:**
```
M_inv = {{11.031566901960783, -9.866943921568629,  -0.16462299647058826},
         {-3.254147380392157,  4.418770392156863,  -0.16462299647058826},
         {-3.6588512862745097, 2.7129230470588235,  1.9459282392156863}}
```

**Scaled XYB parameters:**
```
kScaledXYBOffset = {0.015386134, 0.0, 0.27770459}
kScaledXYBScale  = {22.995788804, 1.183000077, 1.502141333}
kBScale          = 1.0
kYToBRatio       = 1.0
```

**Derived XYB parameters:**
```
kXYBOffset = {kScaledXYBOffset0 + kScaledXYBOffset1,
              kScaledXYBOffset1 - kScaledXYBOffset0 + 1/kScaledXYBScale0,
              kScaledXYBOffset1 + kScaledXYBOffset2}

kXYBScale = {ReciprocialSum(kScaledXYBScale0, kScaledXYBScale1),
             ReciprocialSum(kScaledXYBScale0, kScaledXYBScale1),
             ReciprocialSum(kScaledXYBScale1, kScaledXYBScale2)}
```
Where `ReciprocialSum(r1, r2) = (r1 * r2) / (r1 + r2)`.

### CIELAB Constants (used in ICC tone mapping LUT, `jxl_cms_internal.h:173`)
```
kXn = 0.964212    D50 white point X
kYn = 1.0
kZn = 0.825188    D50 white point Z
kDelta = 6.0 / 29.0   Lab function linearization threshold
```


## Transfer Function Formulas (EXACT - every coefficient)

### sRGB

**Specification formulas** (exact, used in ICC profile generation):
```
Encoded from Linear (OETF):
  if linear <= 0.0031308:
    encoded = 12.92 * linear
  else:
    encoded = 1.055 * linear^(1/2.4) - 0.055

Linear from Encoded (EOTF):
  if encoded <= 0.04045:
    linear = encoded / 12.92
  else:
    linear = ((encoded + 0.055) / 1.055)^2.4
```

**SIMD implementation** (`TF_SRGB`, `transfer_functions-inl.h:215`):
Uses 4/4-degree rational polynomial approximations instead of `pow()`.

`DisplayFromEncoded`: if `x > 0.04045` use rational poly on `x`, else `x / 12.92`.
`EncodedFromDisplay`: if `x > 0.0031308` use rational poly on `sqrt(x)`, else `x * 12.92`.

Rational polynomial coefficients for `DisplayFromEncoded(x)`:
```
p = {2.200248328e-04, 1.043637593e-02, 1.624820318e-01, 7.961564959e-01, 8.210152774e-01}
q = {2.631846970e-01, 1.076976492e+00, 4.987528350e-01, -5.512498495e-02, 6.521209011e-03}
```

Rational polynomial coefficients for `EncodedFromDisplay(sqrt(x))`:
```
p = {-5.135152395e-04, 5.287254571e-03, 3.903842876e-01, 1.474205315e+00, 7.352629620e-01}
q = {1.004519624e-02, 3.036675394e-01, 1.340816930e+00, 9.258482155e-01, 2.424867759e-02}
```

Both use copysign mirroring for negative inputs (extended range / unbounded CMM).

### Perceptual Quantizer (PQ / SMPTE ST 2084)

**Exact scalar formulas** (`TF_PQ_Base`, `transfer_functions.h:90`):
```
DisplayFromEncoded(intensity_target, e):
  x' = |e|^(1/kM2)
  num = max(x' - kC1, 0)
  den = kC2 - kC3 * x'
  d = (num / den)^(1/kM1) * (10000 / intensity_target)
  return copysign(d, e)

EncodedFromDisplay(intensity_target, d):
  x' = (|d| * intensity_target / 10000)^kM1
  num = kC1 + x' * kC2
  den = 1 + x' * kC3
  e = (num / den)^kM2
  return copysign(e, d)
```

**SIMD implementation** (`TF_PQ`, `transfer_functions-inl.h:135`):
Constructor takes `display_intensity_target` (default `kDefaultIntensityTarget = 255`).
Pre-computes scaling factors:
```
display_scaling_factor_to_10000 = intensity_target / 10000
display_scaling_factor_from_10000 = 10000 / intensity_target
```

`DisplayFromEncoded`: 4/4-degree rational polynomial on `x + x*x`:
```
p = {2.62975656e-04, -6.23553089e-03, 7.38602301e-01, 2.64553172e+00, 5.50034862e-01}
q = {4.21350107e+02, -4.28736818e+02, 1.74364667e+02, -3.39078883e+01, 2.67718770e+00}
```
Result multiplied by `display_scaling_factor_from_10000`. Max error 3e-6.

`EncodedFromDisplay`: Two 4/4-degree rational polynomials on `x^0.25`:
- For `x >= 1e-4`:
```
p = {1.351392e-02, -1.095778e+00, 5.522776e+01, 1.492516e+02, 4.838434e+01}
q = {1.012416e+00, 2.016708e+01, 9.263710e+01, 1.120607e+02, 2.590418e+01}
```
- For `x < 1e-4`:
```
plo = {9.863406e-06, 3.881234e-01, 1.352821e+02, 6.889862e+04, -2.864824e+05}
qlo = {3.371868e+01, 1.477719e+03, 1.608477e+04, -4.389884e+04, -2.072546e+05}
```
Max error 7e-7.

### Hybrid Log-Gamma (HLG)

**Scalar OETF** (`TF_HLG_Base::OETF`, `transfer_functions.h:41`):
```
if s <= 1/12:
  encoded = sqrt(3 * s)
else:
  encoded = 0.17883277 * ln(12*s - 0.28466892) + 0.5599107295
```

**Scalar Inverse OETF** (`TF_HLG_Base::InvOETF`, `transfer_functions.h:54`):
```
if e <= 0.5:
  scene = e^2 / 3
else:
  scene = (exp((e - 0.5599107295) / 0.17883277) + 0.28466892) / 12
```

**OOTF**: Identity at 334 nits (gamma = 1.0). The base class uses `OOTF(s) = s`
and `InvOOTF(d) = d`. The actual OOTF is applied separately via `ApplyHlgOotf()`.

**Combined EOTF** (`TF_HLG_Base::DisplayFromEncoded`):
```
DisplayFromEncoded(e) = OOTF(InvOETF(e)) = InvOETF(e)   [since OOTF is identity]
```

**SIMD `EncodedFromDisplay`** (`TF_HLG`, `transfer_functions-inl.h:57`):
```
if |x| <= 1/12:
  magnitude = sqrt(3 * |x|)
else:
  magnitude = kA/ln2 * FastLog2(12*|x| - kB) + kC
  // where kA * kInvLog2e = kA/ln2
```

**SIMD `DisplayFromEncoded`** (`TF_HLG`, `transfer_functions-inl.h:72`):
```
if |x| <= 0.5:
  magnitude = x^2 / 3
else:
  magnitude = FastPow2(x * kHiPow) * kHiMul + kHiAdd
  // where kHiPow = kRA * log2(e), kHiMul = exp(-kC*kRA)/12, kHiAdd = kB/12
```

Max error 5e-7.

### BT.709

**Scalar formula** (`TF_709`, `transfer_functions-inl.h:96`):
```
EncodedFromDisplay(d):
  if d < 0.018:
    encoded = 4.5 * d
  else:
    encoded = 1.099 * d^0.45 - 0.099

DisplayFromEncoded(e):
  if e < 0.081:
    display = e / 4.5
  else:
    display = ((e + 0.099) / 1.099)^(1/0.45)
```

Max error 1e-6 (SIMD version).

### DCI (Gamma 2.6)

Pure power function with no linear segment:
```
EncodedFromDisplay(d) = d^(1/2.6)
DisplayFromEncoded(e) = e^2.6
```
Represented as `TransferFunction::kDCI`. In ICC profiles: `para` type 0 with gamma=2.6.

### FastLinearToSRGB (`transfer_functions-inl.h:279`)

A fast approximation of linear-to-sRGB with max error 1.2e-4.
Uses a 3rd-degree polynomial on the mantissa range [0.25, 0.5]:
```
d1 = 0.059914046 * v + (-0.108894556)
d2 = d1 * v + 0.107963754
pow = d2 * v + 0.018092343
```
Combined with a lookup table of `2^(5/12)` powers indexed by the IEEE 754 exponent
to reconstruct the full `v^(1/2.4)` curve via the identity `v^(1/2.4) = mantissa^(1/2.4) * 2^(exp/2.4)`.

Below `0.0031308`, falls back to `12.92 * v`.


## Tone Mapping Algorithm

### Rec. 2408 Tone Mapper (`Rec2408ToneMapperBase`, `tone_mapping.h:23`)

Maps HDR content from `source_range` to `target_range` (both in nits).
Based on ITU-R BT.2408.

**Construction:**
```
pq_mastering_min = PQ_encode(source_range[0])
pq_mastering_max = PQ_encode(source_range[1])
pq_mastering_range = pq_mastering_max - pq_mastering_min
min_lum = (PQ_encode(target_range[0]) - pq_mastering_min) / pq_mastering_range
max_lum = (PQ_encode(target_range[1]) - pq_mastering_min) / pq_mastering_range
ks = 1.5 * max_lum - 0.5
normalizer = source_range[1] / target_range[1]
inv_target_peak = 1 / target_range[1]
```

**ToneMap algorithm:**
```
1. Compute luminance:
   L = source_range[1] * (red_Y * R + green_Y * G + blue_Y * B)

2. Normalize to PQ mastering range:
   normalized_pq = min(1, (PQ_encode(L) - pq_mastering_min) / pq_mastering_range)

3. Apply Hermite spline if above knee point:
   if normalized_pq < ks:
     e2 = normalized_pq
   else:
     t = (normalized_pq - ks) / (1 - ks)
     e2 = (2*t^3 - 3*t^2 + 1)*ks + (t^3 - 2*t^2 + t)*(1 - ks) + (-2*t^3 + 3*t^2)*max_lum

4. Apply black level lift:
   e3 = min_lum * (1 - e2)^4 + e2

5. Convert back to display light:
   e4 = e3 * pq_mastering_range + pq_mastering_min
   new_luminance = clamp(PQ_decode(e4, intensity=1.0), 0, target_range[1])

6. Scale RGB channels:
   if luminance <= 1e-6:
     channel = new_luminance * inv_target_peak     (cap for near-black)
   else:
     channel *= (new_luminance / luminance) * normalizer
```

**Typical usage in ICC generation:**
- PQ: source `[0, 10000]` nits, target `[0, 250]` nits.
- The `PQ_encode`/`PQ_decode` used here operates at `intensity_target = 1.0` (raw PQ nits).

### HLG OOTF (`HlgOOTF`, `tone_mapping-inl.h:102`)

Applies the Opto-Optical Transfer Function for HLG per BT.2100.

**Gamma computation:**
```
FromSceneLight:
  gamma = 1.2 * pow(1.111, log2(display_luminance / 1000))

ToSceneLight:
  gamma = (1/1.2) * pow(1.111, -log2(display_luminance / 1000))
```

**Application:**
```
luminance = red_Y * R + green_Y * G + blue_Y * B
ratio = min(pow(luminance, gamma - 1), 1e9)
R *= ratio; G *= ratio; B *= ratio
```

OOTF is skipped if `|exponent| < 0.01` (i.e., gamma very close to 1.0).

**In `ApplyHlgOotf` (jxl_cms.cc:857):**
The OOTF is skipped entirely if `intensity_target` is in `[295, 305]`
(since gamma ~1.0 at 300 nits: `1.2 * 1.111^log2(300/1000) = ~1.0`).

### Gamut Mapping (`GamutMap`, `tone_mapping-inl.h:138`)

Desaturates out-of-gamut pixels by mixing with gray at the same luminance.

```
1. Compute luminance from primaries_luminances.

2. For each channel, compute how much gray must be mixed in:
   - gray_mix_saturation: minimum gray to make all components >= 0
   - gray_mix_luminance: minimum gray to make all components <= 1

3. Blend between the two:
   gray_mix = clamp(preserve_saturation * (gray_mix_saturation - gray_mix_luminance)
                    + gray_mix_luminance, 0, 1)

4. Mix: val = gray_mix * (luminance - val) + val

5. Normalize: divide all channels by max(1, max_channel)
```

`preserve_saturation` defaults to `0.1` in the SIMD version.
ICC tone mapping LUT uses `0.3`.


## Color Encoding Signaling

### Bitstream Representation

The `ColorEncoding` struct in `color_encoding_internal.h` is serialized via
the `VisitFields()` visitor pattern. The codestream can carry:

1. **Structured fields**: `ColorSpace`, `WhitePoint`, `Primaries`, `TransferFunction`,
   `RenderingIntent`, plus optional custom xy coordinates and gamma values. These are
   compact enum-based encodings.

2. **Raw ICC profile**: If the color space cannot be represented by the structured
   fields, the full ICC profile is embedded. The `want_icc_` flag controls this.

The decision logic (`DecideIfWantICC`): if an ICC profile can be reconstructed from
the structured fields, only the fields are stored (more compact). Otherwise, the raw
ICC bytes are stored.

### CICP Mapping

JPEG XL uses CICP (Coding-Independent Code Points) values directly:
- `TransferFunction` enum values match CICP TransferCharacteristics (1, 8, 13, 16, 17, 18).
- `Primaries` enum values match CICP ColourPrimaries (1, 9, 11).
- `WhitePoint` enum values match CICP (1=D65, 10=E, 11=DCI).
- Special case: `kColorPrimariesP3_D65 = 12` (DCI-P3 primaries with D65 white, not in the enum).

`ApplyCICP()` (`jxl_cms.cc:928`) maps CICP to internal encoding, requiring
`matrix_coefficients = 0` (identity) and `full_range = 1`.

### ICC Profile Creation

When structured fields are sufficient, libjxl generates ICC profiles on-the-fly
via `MaybeCreateProfile()`. The profile:
- Uses ICC v4.4 (`0x04400000`).
- PCS is XYZ for SDR content, CIELAB for HDR content (when 3D tone mapping LUT is used).
- Includes `cicp` tag for interoperability.
- Uses Bradford chromatic adaptation to D50 for PCS.
- HDR content (PQ/HLG with known primaries): 9x9x9 3D LUT (`mft1` tag) mapping to CIELAB
  with integrated tone mapping and gamut mapping. This is controlled by
  `JXL_ENABLE_3D_ICC_TONEMAPPING` (default 1).

### XYB ICC Profile

The XYB color space gets a special `mAB` (A2B0) tag:
- A curves: linear (no-op)
- CLUT: 2x2x2 grid encoding the corners of the XYB cube
- M curves: `y = (x / kXYBScale[i] - kXYBOffset[i] - cbrt(-kOpsinAbsorbanceBias[i]))^3`
  (parametric curve type 3)
- Matrix: 3x3 from XYB intermediate to XYZ D50
- B curves: linear (no-op)


## CMS Pipeline Architecture

The CMS pipeline processes data in interleaved float format, row by row:

```
Input pixels (nonlinear, e.g. sRGB)
         |
    [Preprocess: libjxl SIMD TF decode]  (kPQ/kHLG/kSRGB -> linear)
         |
    [HLG OOTF forward, if applicable]
         |
    [Channel expand: 1->3 or CMYK invert]
         |
    [skcms_Transform / cmsDoTransform]    (linear src -> linear dst)
    [or memcpy if skip_lcms]
         |
    [Channel contract: 3->1 if gray dest]
         |
    [HLG OOTF inverse, if applicable]
         |
    [Postprocess: libjxl SIMD TF encode]  (linear -> kPQ/kHLG/kSRGB)
         |
Output pixels (nonlinear, target space)
```

The ExtraTF optimization replaces the skcms/LCMS handling of PQ, HLG, and sRGB
transfer functions with libjxl's own SIMD implementations, which use rational
polynomial approximations for higher throughput. The underlying CMS (skcms or LCMS)
only handles the chromatic adaptation / primaries conversion on linear data.

### Unbounded Color Management

All transfer functions use copysign mirroring for negative inputs:
`f(-x) = -f(x)`. This follows the "Unbounded CMM" approach (from LittleCMS)
where inputs can be negative or above 1.0 due to chromatic adaptation.
Functions are extended naturally above 1.0 (no clamping) to preserve
round-trip accuracy.


## Dependencies

### Build-time configuration
- `JPEGXL_ENABLE_SKCMS`: If 1 (default), use skcms. If 0, use LCMS2.
- `JXL_ENABLE_3D_ICC_TONEMAPPING`: If 1 (default), generate 3D LUT ICC profiles for HDR.

### External libraries
- **skcms**: Google's small CMS library (preferred backend)
- **lcms2**: LittleCMS 2 (alternative backend)
- **Highway (hwy)**: SIMD abstraction for transfer function implementations

### Internal dependencies
- `lib/jxl/base/fast_math-inl.h`: `FastLog2f`, `FastPow2f`, `FastPowf` used in SIMD TFs
- `lib/jxl/base/rational_polynomial-inl.h`: `EvalRationalPolynomial` for PQ/sRGB approximations
- `lib/jxl/base/matrix_ops.h`: `Matrix3x3`, `Vector3`, `Mul3x3Matrix`, `Inv3x3Matrix`
- `lib/jxl/base/common.h`: `kDefaultIntensityTarget`, `Clamp1`, `RoundUpTo`
- `lib/jxl/field_encodings.h`: `MakeBit` for enum bitfield definitions

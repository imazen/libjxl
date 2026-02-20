# Serialization (Fields & Visitor Pattern)

## Source Files

- `lib/jxl/field_encodings.h` (134 lines) -- Defines `Fields` base class, `U32Distr`, `U32Enc`, `Val()`, `Bits()`, `BitsOffset()` constructors, `EnumValid()` template, and the `JXL_FIELDS_NAME` macro.
- `lib/jxl/fields.h` (369 lines) -- Declares `BitsCoder`, `U32Coder`, `U64Coder`, `F16Coder` namespaces, the `Bundle` namespace with static Read/Write/Init/AllDefault/CanEncode, the `Visitor` base class, `VisitorBase`, and `ExtensionStates`.
- `lib/jxl/fields.cc` (595 lines) -- Implements `InitVisitor`, `SetDefaultVisitor`, `AllDefaultVisitor`, `ReadVisitor`, `CanEncodeVisitor`, all `Bundle::*` functions, and all coder Read/CanEncode implementations.
- `lib/jxl/enc_fields.h` (41 lines) -- Declares `WriteCodestreamHeaders`, `WriteFrameHeader`, `WriteSizeHeader`, `WriteQuantizerParams`, `WriteImageMetadata`.
- `lib/jxl/enc_fields.cc` (253 lines) -- Implements `WriteVisitor` and all `*Coder::Write` functions, plus `Bundle::Write` and the top-level write wrappers.
- `lib/jxl/headers.h` (97 lines) -- Declares `SizeHeader`, `PreviewHeader`, `AnimationHeader`, aspect ratio system.
- `lib/jxl/headers.cc` (203 lines) -- Implements `SizeHeader::VisitFields`, `PreviewHeader::VisitFields`, `AnimationHeader::VisitFields`, aspect ratio lookup table.
- `lib/jxl/image_metadata.h` (429 lines) -- Declares `BitDepth`, `ExtraChannelInfo`, `OpsinInverseMatrix`, `ToneMapping`, `CustomTransformData`, `ImageMetadata`, `CodecMetadata`.
- `lib/jxl/image_metadata.cc` (488 lines) -- Implements all `VisitFields` for the above, plus `SetAlphaBits`.
- `lib/jxl/frame_header.h` (508 lines) -- Declares `FrameHeader`, `BlendingInfo`, `AnimationFrame`, `Passes`, `YCbCrChromaSubsampling`, `FrameEncoding`, `ColorTransform`, `BlendMode`, `FrameType` enums.
- `lib/jxl/frame_header.cc` (515 lines) -- Implements `FrameHeader::VisitFields`, `BlendingInfo::VisitFields`, `Passes::VisitFields`, etc.
- `lib/jxl/loop_filter.h` (75 lines) -- Declares `LoopFilter` with Gaborish and EPF parameters.
- `lib/jxl/loop_filter.cc` (102 lines) -- Implements `LoopFilter::VisitFields`.

## Key Types

### `Fields` (base class, `field_encodings.h:32`)

Abstract base class for all serializable bundles. Two virtual methods:
- `VisitFields(Visitor*)` -- the single function that defines the serialization schema.
- `Name()` -- debug-only, returns the struct name (enabled by `JXL_FIELDS_NAME` macro, gated on `JXL_IS_DEBUG_BUILD`).

Every struct that participates in serialization inherits from `Fields`.

### `Visitor` (abstract, `fields.h:190`)

Virtual interface through which `VisitFields` interacts with the serialization system. The same `VisitFields` function serves reading, writing, initialization, validation, and size computation -- the Visitor subclass determines the operation.

Pure virtual methods:
- `Visit(Fields*)` -- entry point, calls `fields->VisitFields(this)`.
- `Bool(bool default_value, bool* value)` -- 1-bit boolean.
- `U32(U32Enc, uint32_t default_value, uint32_t* value)` -- variable-length u32.
- `Bits(size_t bits, uint32_t default_value, uint32_t* value)` -- fixed-width raw bits.
- `U64(uint64_t default_value, uint64_t* value)` -- variable-length u64.
- `F16(float default_value, float* value)` -- IEEE 754 binary16.
- `BeginExtensions(uint64_t* extensions)` -- start of extension region.
- `EndExtensions()` -- end of extension region.

Non-pure virtual methods with default implementations:
- `Conditional(bool condition)` -- returns `condition` by default; overridden by `InitVisitor` to always return `true` (so all fields get initialized regardless of conditions).
- `AllDefault(const Fields&, bool* all_default)` -- reads/writes the all_default bit, returns true (skip) if all_default is true. Overridden by `InitVisitor` and `AllDefaultVisitor`.
- `SetDefault(Fields*)` -- no-op by default; overridden by `ReadVisitor` to call `Bundle::SetDefault`.
- `VisitNested(Fields*)` -- delegates to `Visit` by default; overridden by `InitVisitor` to skip (nested bundles init themselves in their constructors).
- `IsReading()` -- returns `false` by default; overridden by `ReadVisitor` to return `true`.

Helper (non-virtual):
- `U32(d0, d1, d2, d3, default_value, value)` -- constructs `U32Enc` from four `U32Distr` and delegates.
- `Enum(default_value, value)` -- encodes enums as `U32(Val(0), Val(1), BitsOffset(4,2), BitsOffset(6,18))`, followed by `EnumValid(*value)` validation. This gives: `00`->0, `01`->1, `10xxxx`->2..17, `11yyyyyy`->18..81.

### `VisitorBase` (`fields.h:294`, `fields_internal` namespace)

Intermediate class between `Visitor` and the concrete visitors. Provides:
- `Visit(Fields*)`: the single call site for `fields->VisitFields(this)`. Manages a depth counter and `ExtensionStates` stack. Asserts that if `BeginExtensions` was called, `EndExtensions` was also called.
- `VisitConst(const Fields&)`: const-casts and delegates to `Visit`.
- `Bool()`: default implementation that delegates to `Bits(1, ...)`.
- `BeginExtensions()`: reads/writes the extensions u64, then calls `extension_states_.Begin()`.
- `EndExtensions()`: calls `extension_states_.End()`.

### `ExtensionStates` (`fields.h:253`)

A stack of three-state machines (not-begun, active, ended) packed into two `uint64_t` bitmasks (`begun_` and `ended_`). Supports nesting up to 64 levels (one bit per level). Used to enforce that extensions are properly bracketed.

### Concrete Visitor Subclasses (all in `fields.cc` or `enc_fields.cc`)

1. **`InitVisitor`** (`fields.cc:29`): Sets every field to its default value. `Conditional()` always returns `true` so all branches are visited. `AllDefault()` initializes the all_default field but returns `false` to prevent skipping. `VisitNested()` is a no-op because nested bundles self-initialize in their constructors.

2. **`SetDefaultVisitor`** (`fields.cc:76`): Like `InitVisitor` but IS recursive to nested fields (uses the default `VisitNested` which calls `Visit`).

3. **`AllDefaultVisitor`** (`fields.cc:116`): Checks if all fields equal their defaults. Tracks `all_default_` flag via `&=` comparison. `AllDefault()` returns `false` to avoid short-circuiting. For `F16`, uses epsilon comparison (`1E-6f`). For `Bool`, inherits from `VisitorBase` which delegates through `Bits`.

4. **`ReadVisitor`** (`fields.cc:155`): Reads field values from a `BitReader`. Ignores default values. Tracks `ok_` for non-byte errors (like NaN/Inf in F16) and `enough_bytes_` separately. Key override: `SetDefault()` calls `Bundle::SetDefault(fields)`, enabling the all_default shortcut during reading. `BeginExtensions` reads per-extension bit counts. `EndExtensions` skips any remaining unread extension bits (forward compatibility).

5. **`CanEncodeVisitor`** (`fields.cc:275`): Computes total encoded bits without writing. Accumulates `encoded_bits_`. `AllDefault()` calls `Bundle::AllDefault(fields)` to compute the actual all_default value, then encodes the bit. `BeginExtensions` snapshots `encoded_bits_` for later extension-size computation. `GetSizes()` returns extension_bits and total_bits, adding the size of the extension size fields themselves.

6. **`WriteVisitor`** (`enc_fields.cc:27`): Writes field values to a `BitWriter`. Takes `extension_bits` (pre-computed by `CanEncodeVisitor`). `BeginExtensions` writes the extension sizes (currently all bits ascribed to first extension). Delegates to `BitsCoder::Write`, `U32Coder::Write`, etc.

### `Bundle` namespace (`fields.h:151`)

Static functions that orchestrate the visitor pattern:

- **`Init(Fields*)`**: Creates `InitVisitor`, calls `Visit`. Called in every bundle's constructor.
- **`SetDefault(Fields*)`**: Creates `SetDefaultVisitor`, calls `Visit`. Recursive to nested fields.
- **`AllDefault(const Fields&)`**: Creates `AllDefaultVisitor`, visits, returns `visitor.AllDefault()`.
- **`CanEncode(const Fields&, size_t* extension_bits, size_t* total_bits)`**: Creates `CanEncodeVisitor`, visits, calls `GetSizes`.
- **`Read(BitReader*, Fields*)`**: Creates `ReadVisitor`, visits, checks `OK()`.
- **`CanRead(BitReader*, Fields*)`**: Like `Read` but only checks if enough bytes exist (returns true even for content errors).
- **`Write(const Fields&, BitWriter*, LayerType, AuxOut*)`** (in `enc_fields.cc:85`): First calls `CanEncode` to get exact bit count, then `writer->WithMaxBits(total_bits, ...)` to pre-allocate, then creates `WriteVisitor(extension_bits, writer)` and visits.

### `U32Distr` (`field_encodings.h:44`)

A compact representation of one of four distributions in a U32 encoding. Packed into a single `uint32_t d`:
- If bit 31 is set (`kDirect = 0x80000000`): direct value, `Direct() = d & 0x7FFFFFFF`.
- Otherwise: bits field = `(d & 0x1F) + 1` (range 1..32), offset = `(d >> 5) & 0x3FFFFFF`.

### `U32Enc` (`field_encodings.h:76`)

An array of four `U32Distr`, selected by a 2-bit prefix in the bitstream.

### Key Header Structs

**`SizeHeader`** (`headers.h:28`): Compact image dimensions.
- `small_` (bool, default `false`): if true, dimensions <= 256 and divisible by 8.
- `ysize_div8_minus_1_` (5 bits, default 0): y dimension / 8 - 1, only if small.
- `ysize_` (U32, default 1): full y dimension, only if !small. Encoding: `BitsOffset(9,1)|BitsOffset(13,1)|BitsOffset(18,1)|BitsOffset(30,1)`.
- `ratio_` (3 bits, default 0): aspect ratio index (0=custom, 1=1:1, 2=12:10, 3=4:3, 4=3:2, 5=16:9, 6=5:4, 7=2:1).
- `xsize_div8_minus_1_` (5 bits, default 0): only if ratio==0 && small.
- `xsize_` (U32, default 1): only if ratio==0 && !small. Same encoding as ysize_.

**`PreviewHeader`** (`headers.h:54`): Similar to SizeHeader but for smaller previews.
- `div8_` (bool, default `false`): dimensions divisible by 8.
- `ysize_div8_` (U32, default 1): `Val(16)|Val(32)|BitsOffset(5,1)|BitsOffset(9,33)`. Only if div8.
- `ysize_` (U32, default 1): `BitsOffset(6,1)|BitsOffset(8,65)|BitsOffset(10,321)|BitsOffset(12,1345)`. Only if !div8.
- `ratio_` (3 bits, default 0): same aspect ratio table as SizeHeader.
- `xsize_div8_` / `xsize_`: same encodings as ysize variants, conditional on ratio==0.

**`AnimationHeader`** (`headers.h:77`):
- `tps_numerator` (U32, default 1): `Val(100)|Val(1000)|BitsOffset(10,1)|BitsOffset(30,1)`.
- `tps_denominator` (U32, default 1): `Val(1)|Val(1001)|BitsOffset(8,1)|BitsOffset(10,1)`.
- `num_loops` (U32, default 0): `Val(0)|Bits(3)|Bits(16)|Bits(32)`.
- `have_timecodes` (bool, default `false`).

**`ImageMetadata`** (`image_metadata.h:201`): The main codestream metadata.
- `all_default` (bool) -- all-default shortcut bit.
- `extra_fields` (bool, default `false`): gates orientation, preview, animation, intrinsic size, tone mapping.
- `orientation` (3 bits, default 0, stored as orientation-1): EXIF 1-8. Only if extra_fields.
- `have_intrinsic_size` (bool, default `false`): only if extra_fields.
- `have_preview` (bool, default `false`): only if extra_fields.
- `have_animation` (bool, default `false`): only if extra_fields.
- `bit_depth` (nested `BitDepth`): always present.
- `modular_16_bit_buffer_sufficient` (bool, default `true`).
- `num_extra_channels` (U32, default 0): `Val(0)|Val(1)|BitsOffset(4,2)|BitsOffset(12,1)`.
- `extra_channel_info` (vector of `ExtraChannelInfo`): visited if num_extra_channels != 0.
- `xyb_encoded` (bool, default `true`).
- `color_encoding` (nested `ColorEncoding`): always present.
- `tone_mapping` (nested `ToneMapping`): only if extra_fields.
- `extensions` (u64): extension mechanism.

**`BitDepth`** (`image_metadata.h:83`):
- `floating_point_sample` (bool, default `false`).
- If integer: `bits_per_sample` (U32, default 8): `Val(8)|Val(10)|Val(12)|BitsOffset(6,1)`. Valid range: 1..31.
- If float: `bits_per_sample` (U32, default 32): `Val(32)|Val(16)|Val(24)|BitsOffset(6,1)`. `exponent_bits_per_sample` (4 bits, default 7, stored with offset -1): valid range 2..8, mantissa bits must be 2..23.

**`ExtraChannelInfo`** (`image_metadata.h:113`):
- `all_default` (bool) -- all-default shortcut.
- `type` (Enum `ExtraChannel`, default `kAlpha`).
- `bit_depth` (nested `BitDepth`).
- `dim_shift` (U32, default 0): `Val(0)|Val(3)|Val(4)|BitsOffset(3,1)`. Max: `1<<dim_shift <= 8`.
- `name` (string via `VisitNameString`).
- `alpha_associated` (bool, default `false`): only if type==kAlpha.
- `spot_color[4]` (F16, default 0): only if type==kSpotColor.
- `cfa_channel` (U32, default 1): only if type==kCFA. `Val(1)|Bits(2)|BitsOffset(4,3)|BitsOffset(8,19)`.

**`ToneMapping`** (`image_metadata.h:149`):
- `all_default` (bool) -- all-default shortcut.
- `intensity_target` (F16, default `kDefaultIntensityTarget`): must be > 0.
- `min_nits` (F16, default 0.0): must be in [0, intensity_target].
- `relative_to_max_display` (bool, default `false`).
- `linear_below` (F16, default 0.0): must be >= 0; if relative, must be <= 1.0.

**`FrameHeader`** (`frame_header.h:330`):
- `all_default` (bool) -- all-default shortcut. If true, entire frame header is 1 bit.
- `frame_type` (FrameType, encoded via `VisitFrameType`): `Val(0)|Val(1)|Val(2)|Val(3)`, default `kRegularFrame` (0).
- `encoding` (FrameEncoding): encoded as bool `is_modular`, default `false` (VarDCT).
- `flags` (u64, default 0).
- `color_transform`: if xyb_encoded, forced to `kXYB` (not serialized). Otherwise, encoded as bool `alternate` (default `false`): false->kNone, true->kYCbCr.
- `chroma_subsampling` (nested): only if YCbCr and no DC frame.
- `upsampling` (U32, default 1): `Val(1)|Val(2)|Val(4)|Val(8)`. Only if no DC frame.
- `extra_channel_upsampling` (vector of U32): per extra channel, same encoding. Only if no DC frame and extra channels exist.
- `group_size_shift` (2 bits, default 1): only if modular.
- `x_qm_scale` (3 bits, default 3), `b_qm_scale` (3 bits, default 2): only if VarDCT+XYB.
- `passes` (nested `Passes`): only if not kReferenceOnly.
- `dc_level` (U32, default 1): `Val(1)|Val(2)|Val(3)|Val(4)`. Only if kDCFrame.
- `custom_size_or_origin` (bool, default `false`): only if not kDCFrame.
- Frame origin x0, y0 (U32, packed signed, default 0): only if custom + regular/skip-progressive. Encoding: `Bits(8)|BitsOffset(11,256)|BitsOffset(14,2304)|BitsOffset(30,18688)`.
- Frame size xsize, ysize (U32, default 0): same encoding as origin.
- `blending_info` (nested): only if regular/skip-progressive.
- `extra_channel_blending_info` (vector of nested `BlendingInfo`): one per extra channel.
- `animation_frame` (nested): only if have_animation.
- `is_last` (bool, default `true`): only for regular/skip-progressive frames.
- `save_as_reference` (U32, default 0): `Val(0)|Val(1)|Val(2)|Val(3)`. Only if not DC frame and not last.
- `save_before_color_transform` (bool): conditional on frame type and blending mode.
- `name` (string via `VisitNameString`).
- `loop_filter` (nested `LoopFilter`).
- `extensions` (u64): extension mechanism.

**`LoopFilter`** (`loop_filter.h:20`):
- `all_default` (bool) -- all-default shortcut.
- `gab` (bool, default `true`): Gaborish convolution enabled.
- `gab_custom` (bool, default `false`): only if gab.
- Gaborish weights (6 x F16): only if gab_custom.
- `epf_iters` (2 bits, default 2): EPF iterations (0=disabled, 1-3).
- EPF parameters: `epf_sharp_custom`, `epf_weight_custom`, `epf_sigma_custom` with associated F16 values. Some conditional on `!nonserialized_is_modular`.
- `epf_sigma_for_modular` (F16, default 1.0): only if modular and epf_iters > 0.
- `extensions` (u64): extension mechanism.

**`Passes`** (`frame_header.h:262`):
- `num_passes` (U32, default 1): `Val(1)|Val(2)|Val(3)|BitsOffset(3,4)`. Max: `kMaxNumPasses`.
- If num_passes != 1:
  - `num_downsample` (U32, default 0): `Val(0)|Val(1)|Val(2)|BitsOffset(1,3)`. Must be <= num_passes.
  - `shift[i]` (2 bits, default 0): for i in 0..num_passes-2. Last is implicitly 0.
  - `downsample[i]` (U32): `Val(1)|Val(2)|Val(4)|Val(8)`. Must be decreasing.
  - `last_pass[i]` (U32): `Val(0)|Val(1)|Val(2)|Bits(3)`. Must be increasing.

**`BlendingInfo`** (`frame_header.h:212`):
- `mode` (BlendMode via `VisitBlendMode`): `Val(0)|Val(1)|Val(2)|BitsOffset(2,3)`, default kReplace (0).
- `alpha_channel` (U32, default 0): `Val(0)|Val(1)|Val(2)|BitsOffset(3,3)`. Only if blend/alpha-weighted-add and extra channels exist.
- `clamp` (bool, default `false`): only if blend/alpha-weighted-add/mul with extra channels.
- `source` (U32, default 0): `Val(0)|Val(1)|Val(2)|Val(3)`. Only if mode != kReplace or partial frame.

### `nonserialized_*` Fields

Many structs carry `nonserialized_*` fields that are NOT read from or written to the bitstream. They are metadata injected before (de)serialization to provide context for conditional fields:
- `FrameHeader::nonserialized_metadata` -- pointer to `CodecMetadata`, provides `xyb_encoded`, extra channel info, animation flag, image dimensions.
- `FrameHeader::nonserialized_is_preview` -- whether this frame is the preview frame.
- `BlendingInfo::nonserialized_num_extra_channels` -- controls whether alpha_channel is serialized.
- `BlendingInfo::nonserialized_is_partial_frame` -- controls whether source is serialized.
- `AnimationFrame::nonserialized_metadata` -- provides `have_animation` and `have_timecodes`.
- `LoopFilter::nonserialized_is_modular` -- gates VarDCT-only EPF parameters.
- `CustomTransformData::nonserialized_xyb_encoded` -- gates opsin inverse matrix.
- `ImageMetadata::nonserialized_only_parse_basic_info` -- early exit after basic fields.

These must be set correctly before calling `Bundle::Read` or `Bundle::Write`. The decoder sets them from previously-decoded headers; the encoder sets them from the in-memory metadata.

## Key Functions

### The `VisitFields` Pattern

Every serializable struct implements `VisitFields(Visitor*)`. This single function defines the entire schema -- field order, types, default values, conditions, and nested bundles. The same function is called by all six visitor types (Init, SetDefault, AllDefault, Read, CanEncode, Write).

Typical pattern:
```cpp
Status MyStruct::VisitFields(Visitor* visitor) {
    // 1. Optional: all-default shortcut
    if (visitor->AllDefault(*this, &all_default)) {
        visitor->SetDefault(this);
        return true;
    }

    // 2. Visit fields in order
    JXL_QUIET_RETURN_IF_ERROR(visitor->Bool(false, &my_bool));
    JXL_QUIET_RETURN_IF_ERROR(visitor->U32(Val(0), Val(1), Bits(4), Bits(8),
                                           0, &my_u32));

    // 3. Conditional fields
    if (visitor->Conditional(my_bool)) {
        JXL_QUIET_RETURN_IF_ERROR(visitor->F16(1.0f, &my_float));
    }

    // 4. Nested bundles
    JXL_RETURN_IF_ERROR(visitor->VisitNested(&nested_bundle));

    // 5. Extensions
    JXL_QUIET_RETURN_IF_ERROR(visitor->BeginExtensions(&extensions));
    // future extension fields would go here
    return visitor->EndExtensions();
}
```

### `VisitNameString` (`frame_header.h:35`)

Serializes a UTF-8 string. Length is encoded as `U32(Val(0), Bits(4), BitsOffset(5,16), BitsOffset(10,48))` -- allowing 0, 1-16, 16-47, 48-1071 bytes. Each character is 8 raw bits.

### `Conditional()` control flow

`Conditional` does NOT use if/else -- both branches must be visited with complementary conditions:
```cpp
if (visitor->Conditional(some_flag)) {
    visitor->Bits(8, 0, &field_a);
}
if (visitor->Conditional(!some_flag)) {
    visitor->Bits(16, 0, &field_b);
}
```
WARNING: an `else` branch would prevent `InitVisitor` (which returns `true` for all conditions) from initializing `field_b`.

### Enum encoding

Via `Visitor::Enum()`:
- `00` -> 0
- `01` -> 1
- `10` + 4 bits -> 2..17
- `11` + 6 bits -> 18..81

After decoding, `EnumValid()` checks the value against a bitmask returned by `EnumBits(Enum())`. Each enum type provides `EnumBits()` and `EnumName()` overloads.

## Constants

### SizeHeader aspect ratios (index 1-7)
| Index | Ratio | Example |
|-------|-------|---------|
| 1 | 1:1 | Square |
| 2 | 12:10 | |
| 3 | 4:3 | Camera |
| 4 | 3:2 | Mobile camera |
| 5 | 16:9 | Camera/display |
| 6 | 5:4 | |
| 7 | 2:1 | |

Index 0 means the x dimension is stored explicitly.

### Codestream signature
- `0xFF` followed by `0x0A` (the `kCodestreamMarker`). The `0x0A` is line feed, chosen because it's reserved by JPEG (ISO/IEC 10918-1) and would cause text-mode-opened files to be rejected if the marker byte changes.

### Frame flags
| Flag | Value | Meaning |
|------|-------|---------|
| kNoise | 1 | Inject noise |
| kPatches | 2 | Overlay patches |
| kSplines | 16 | Overlay splines |
| kUseDcFrame | 32 | Use DC frame (implies kSkipAdaptiveDCSmoothing) |
| kSkipAdaptiveDCSmoothing | 128 | Skip adaptive DC smoothing |

### Default values for key fields
- `FrameHeader::all_default` = true (entire header can be 1 bit)
- `FrameHeader::encoding` = VarDCT (is_modular = false)
- `FrameHeader::frame_type` = kRegularFrame (0)
- `FrameHeader::flags` = 0
- `FrameHeader::upsampling` = 1
- `FrameHeader::group_size_shift` = 1 (modular groups are 256x256)
- `FrameHeader::x_qm_scale` = 3
- `FrameHeader::b_qm_scale` = 2
- `FrameHeader::is_last` = true
- `ImageMetadata::xyb_encoded` = true
- `ImageMetadata::bit_depth.bits_per_sample` = 8 (integer)
- `ImageMetadata::modular_16_bit_buffer_sufficient` = true
- `ImageMetadata::orientation` = 1 (identity)
- `ToneMapping::intensity_target` = `kDefaultIntensityTarget` (255.0)
- `LoopFilter::gab` = true
- `LoopFilter::epf_iters` = 2

### Bundle::kMaxExtensions = 64
Maximum number of extension bits (matches u64 width).

## The U32 Encoding Scheme

The `U32Coder` is the most distinctive encoding primitive in libjxl. It encodes a `uint32_t` using a configurable 4-entry lookup table.

### Structure

Each `U32Enc` contains four `U32Distr` entries, selected by a 2-bit prefix:
- `00` -> distribution 0
- `01` -> distribution 1
- `10` -> distribution 2
- `11` -> distribution 3

Each distribution is either:
- **Direct**: `Val(v)` -- the value is exactly `v`, no extra bits needed. Total: 2 bits.
- **Offset+Bits**: `BitsOffset(n, offset)` -- read `n` extra bits, add `offset`. Total: 2 + n bits. `Bits(n)` is shorthand for `BitsOffset(n, 0)`.

### Encoding algorithm (`U32Coder::ChooseSelector`, `fields.cc:454`)

To encode value `v`:
1. Try all 4 selectors.
2. For direct selectors: if `Direct() == v`, use it (always optimal at 2 bits).
3. For offset selectors: check if `offset <= v < offset + 2^extra_bits`. If so, cost is `2 + extra_bits`.
4. Choose the selector with fewest total bits.
5. Write selector (2 bits), then for non-direct: write `v - offset` in `extra_bits` bits.

### Decoding algorithm (`U32Coder::Read`, `fields.cc:444`)

1. Read 2-bit selector.
2. Look up `U32Distr` for that selector.
3. If direct: return `Direct()`.
4. Otherwise: read `ExtraBits()` bits, add `Offset()`, return result.

### Common U32 distributions used in libjxl

**Image dimensions** (`SizeHeader`): `BitsOffset(9,1) | BitsOffset(13,1) | BitsOffset(18,1) | BitsOffset(30,1)`
- `00` + 9 bits: 1..512
- `01` + 13 bits: 1..8192
- `10` + 18 bits: 1..262144
- `11` + 30 bits: 1..1073741824

**Passes count**: `Val(1) | Val(2) | Val(3) | BitsOffset(3,4)`
- `00`: 1, `01`: 2, `10`: 3, `11` + 3 bits: 4..11

**Extra channels count**: `Val(0) | Val(1) | BitsOffset(4,2) | BitsOffset(12,1)`
- `00`: 0, `01`: 1, `10` + 4 bits: 2..17, `11` + 12 bits: 1..4096

## The U64 Encoding Scheme (`U64Coder`)

A variable-length encoding for 64-bit values. Uses a 2-bit selector:

| Selector | Payload | Range | Total bits |
|----------|---------|-------|------------|
| `00` | none | 0 | 2 |
| `01` | 4 bits | 1..16 | 6 |
| `10` | 8 bits | 17..272 | 10 |
| `11` | 12 bits + varint | 273..2^64-1 | 14+ |

For selector 3 (varint): After the initial 12-bit group, additional 8-bit groups follow, each preceded by a 1-bit continuation flag. When `shift == 60`, the final group is 4 bits (filling the remaining bits of a u64). A 0 continuation bit terminates the sequence.

Maximum encoded bits: `2 + 12 + 6*(8+1) + (4+1) = 73` bits.

## The F16 Encoding Scheme (`F16Coder`)

IEEE 754 binary16 (half precision). Always exactly 16 bits. NaN and Infinity are rejected (return failure). Subnormals are supported.

Encoding (`enc_fields.cc:168`): Convert float32 to float16 by adjusting exponent bias (127 -> 15) and truncating mantissa (23 -> 10 bits). Values too large (exp > 15) fail. Values tiny enough (exp < -24) are written as zero.

Decoding (`fields.cc:550`): Read 16 bits, decompose into sign/exponent/mantissa. For subnormals (biased_exp == 0), compute via `(1/16384) * (mantissa/1024)`. For normals, reconstruct float32 bits directly.

Range: approximately [-65504, 65504].

## Algorithm Details

### Serialization flow (writing a header)

1. Caller invokes `Bundle::Write(fields, writer, layer, aux_out)` (enc_fields.cc:85).
2. `Bundle::CanEncode` runs first: creates `CanEncodeVisitor`, visits fields, computes `extension_bits` and `total_bits`.
3. `writer->WithMaxBits(total_bits, ...)` pre-allocates exactly enough space.
4. `WriteVisitor(extension_bits, writer)` is created and `VisitConst(fields)` is called.
5. Each field's value is written by the corresponding coder (`BitsCoder::Write`, `U32Coder::Write`, etc.).
6. If the `all_default` shortcut fires (all fields are defaults): only 1 bit (`true`) is written, and the function returns immediately.
7. Otherwise: `all_default` bit is written as `false`, then every field is written in order.
8. For extension fields: `BeginExtensions` writes the extensions u64, then per-extension bit counts. `EndExtensions` is a no-op for writing.

### Deserialization flow (reading a header)

1. Caller invokes `Bundle::Read(reader, fields)` (fields.cc:392).
2. Creates `ReadVisitor(reader)`, calls `Visit(fields)`.
3. `Visit` calls `fields->VisitFields(this)`.
4. First: `AllDefault` reads 1 bit. If `true`, calls `SetDefault(fields)` (which runs `SetDefaultVisitor`) and returns. The entire header was 1 bit.
5. If `false`: each field is read from the bitstream by the corresponding coder.
6. `Conditional(expr)`: evaluates `expr` using previously-read field values. If false, the field is not read (it retains its init-time default).
7. `VisitNested(&bundle)`: recursively calls `Visit(bundle)` which calls `bundle->VisitFields(this)`.
8. `BeginExtensions`: reads the extensions u64. For each set bit, reads a u64 giving the bit count for that extension.
9. `EndExtensions`: computes how many extension bits were actually consumed. Skips any remaining bits (forward compatibility -- old decoders skip unknown extensions).
10. After visiting, `visitor.OK()` is checked for non-byte errors (e.g., F16 NaN).

### Initialization flow

1. Every bundle's constructor calls `Bundle::Init(this)`.
2. `InitVisitor` visits the `VisitFields` function:
   - All field visit methods just set `*value = default_value`.
   - `Conditional()` always returns `true` -- all branches are taken, all fields initialized.
   - `AllDefault()` initializes the all_default field but returns `false` (don't skip).
   - `VisitNested()` is a no-op -- nested bundles initialize themselves in their own constructors.

### Validation

Validation is distributed across the `VisitFields` implementations, not centralized. It occurs at multiple points:

1. **During reading**: After reading a field value, inline checks in `VisitFields` validate ranges:
   ```cpp
   if (dim_shift > 3) return JXL_FAILURE("dim_shift too large");
   ```
2. **Enum validation**: `Visitor::Enum()` calls `EnumValid(*value)` after decoding, checking the value against the registered bitmask.
3. **Cross-field validation**: Some fields are validated against each other:
   - `num_downsample <= num_passes`
   - `downsample` sequence must be decreasing
   - `last_pass` sequence must be increasing
   - `ec_upsampling >= upsampling`
   - `alpha_channel < nonserialized_num_extra_channels`
4. **During encoding**: `CanEncodeVisitor` calls each coder's `CanEncode`, which checks that values fit in the declared bit widths. `U32Coder::ChooseSelector` fails if no selector can represent the value.
5. **F16 validation**: NaN and Inf are rejected on both read and write. The max encodable magnitude is 65504.
6. **ToneMapping**: `intensity_target > 0`, `0 <= min_nits <= intensity_target`, `linear_below >= 0`.
7. **BitDepth**: integer `bits_per_sample` must be <= 31; float `exponent_bits_per_sample` must be 2..8, mantissa 2..23.

### The AllDefault Shortcut

The most important optimization: if every serialized field in a bundle equals its default, the entire bundle is encoded as a single `1` bit. This is how the common case of a default `FrameHeader` becomes just 1 bit in the bitstream.

The flow:
1. `VisitFields` starts with `if (visitor->AllDefault(*this, &all_default))`.
2. **ReadVisitor**: `AllDefault` reads 1 bit into `all_default`. If `true`, `SetDefault` is called (restoring all defaults), and VisitFields returns early.
3. **WriteVisitor** (via `VisitorBase`): `AllDefault` writes 1 bit. If `true` (pre-computed by `CanEncodeVisitor`), returns early.
4. **CanEncodeVisitor**: `AllDefault` calls `Bundle::AllDefault(fields)` to compute whether all fields really are defaults. Sets `*all_default` to the result, encodes 1 bit. If true, returns early (only 1 bit encoded).
5. **InitVisitor**: sets `all_default = true` (default) but returns `false` (don't skip other fields).
6. **AllDefaultVisitor**: returns `false` (don't skip, need to check all fields).

Important: `AllDefault` checks only serialized fields. `nonserialized_*` fields are ignored. Also, if `extensions != 0`, `AllDefault` is `false` because the extensions field is serialized.

### Extension Mechanism

The extension mechanism provides forward compatibility. Every bundle that may be extended includes an `extensions` u64 field and brackets future fields with `BeginExtensions`/`EndExtensions`.

**Writing with extensions:**
1. `BeginExtensions` writes the `extensions` u64 (0 if no extensions).
2. If extensions != 0, writes per-extension bit counts (u64 each). Currently all bits are ascribed to the first extension; others get 0.
3. Extension field values are written normally.
4. `EndExtensions` is a no-op.

**Reading with extensions:**
1. `BeginExtensions` reads the `extensions` u64.
2. If extensions != 0, reads per-extension bit counts. Records `pos_after_ext_size_` (total bits consumed after reading sizes) and `total_extension_bits_`.
3. Extension fields known to this decoder version are read normally.
4. `EndExtensions` computes `remaining_bits = (pos_after_ext_size_ + total_extension_bits_) - bits_read`. Skips any remaining bits (these are future extensions this decoder doesn't know about).

**Forward compatibility**: An old decoder encountering a new bitstream with unknown extensions will read the extension sizes, skip the unknown bits, and continue. The decoder only processes extensions it knows about.

**Backward compatibility**: A new decoder encountering an old bitstream with `extensions == 0` skips the entire extension region (no sizes to read, no bits to skip).

### The Codestream Header Structure

Written by `WriteCodestreamHeaders` (enc_fields.cc:209):
1. Signature: `0xFF 0x0A` (16 bits, raw).
2. `SizeHeader` (image dimensions).
3. `ImageMetadata` (bit depth, color, extra channels, orientation, animation, tone mapping, extensions).
4. `CustomTransformData` (opsin inverse matrix, upsampling weights).

Then per frame:
5. `FrameHeader` (frame type, encoding, flags, passes, blending, loop filter, extensions).

## Dependencies

- `lib/jxl/base/status.h` -- `Status`, `JXL_RETURN_IF_ERROR`, `JXL_FAILURE`, `JXL_NOT_ENOUGH_BYTES`.
- `lib/jxl/base/bits.h` -- `Num0BitsAboveMS1Bit`, `Num0BitsBelowLS1Bit_Nonzero`.
- `lib/jxl/dec_bit_reader.h` -- `BitReader` (reading side).
- `lib/jxl/enc_bit_writer.h` -- `BitWriter` (writing side, encoder-only).
- `lib/jxl/pack_signed.h` -- `PackSigned`/`UnpackSigned` for signed integer encoding.
- `lib/jxl/color_encoding_internal.h` -- `ColorEncoding` (nested bundle in `ImageMetadata`).
- `lib/jxl/cms/opsin_params.h` -- Default opsin matrix values.
- `hwy/base.h` -- `PopCount` used in `CanEncodeVisitor::GetSizes`.

## Open Questions

1. **Per-extension bit counts**: The current encoder ascribes all extension bits to the first extension and writes 0 for others (TODO comment in both `CanEncodeVisitor::GetSizes` and `WriteVisitor::BeginExtensions`). If multiple extensions are ever used simultaneously, the API would need to be extended to pass per-extension bit arrays.

2. **AllDefault for complex fields**: `AllDefaultVisitor` compares F16 with epsilon `1E-6f`, but `Bool` goes through `Bits(1, ...)` which uses exact `==`. If a float field has accumulated rounding from a codec pipeline, it might not round-trip perfectly through `AllDefault`, potentially bloating the bitstream by 1 header's worth.

3. **Enum range**: The `Enum()` encoding supports up to value 81 (`11` + 6 bits + offset 18). Enum values above 63 would also fail `EnumValid` since it checks bit position in a u64 bitmask. The practical limit is 63.

4. **VisitFields is non-const for writing**: Even `WriteVisitor` and `CanEncodeVisitor` call `VisitFields` on a non-const Fields pointer (via `VisitConst` which const-casts). This is because `VisitFields` takes `uint32_t*` pointers to fields (needed for reading). The `CanEncodeVisitor` does modify `all_default` fields as a side effect, hence the comment "C is not modified except the `all_default` field."

5. **MaxBits pre-allocation**: `Bundle::Write` computes exact bit count via `CanEncode` before writing, then calls `WithMaxBits`. This means every write does two full traversals of the field tree. For small headers this is negligible; for deeply nested or repeated structures (many extra channels, many passes), the double traversal might matter.

6. **`nonserialized_*` initialization responsibility**: The caller must set `nonserialized_*` fields before calling `Bundle::Read` or `Bundle::Write`. There is no compile-time enforcement of this. If forgotten, conditions that depend on `nonserialized_metadata` (which defaults to `nullptr`) will take the wrong branch, leading to silent corruption or crashes. This is a common source of bugs when adding new frame types or reading paths.

# Encoder Pipeline Details

## Source Files

| File | Purpose |
|------|---------|
| `lib/jxl/enc_frame.cc` | Main frame encoding entry point, orchestrates the entire pipeline |
| `lib/jxl/enc_frame.h` | Public `EncodeFrame` signature |
| `lib/jxl/enc_group.cc` | Per-group coefficient computation and tokenized coefficient writing |
| `lib/jxl/enc_group.h` | Group encoding public API |
| `lib/jxl/enc_progressive_split.h` | `ProgressiveSplitter` class and `PassDefinition`/`ProgressiveMode` structs |
| `lib/jxl/enc_progressive_split.cc` | `SplitACCoefficients` implementation |
| `lib/jxl/frame_header.h` | `FrameHeader`, `Passes`, `FrameEncoding`, `ColorTransform`, `BlendMode` |
| `lib/jxl/frame_dimensions.h` | `FrameDimensions` struct (groups, blocks, DC groups) |
| `lib/jxl/passes_state.h` | `PassesSharedState`, `ImageFeatures` |
| `lib/jxl/enc_cache.h` | `PassesEncoderState` (main encoder state struct) |
| `lib/jxl/toc.h` | TOC distribution, `AcGroupIndex()`, `NumTocEntries()`, `ReadToc()` |
| `lib/jxl/toc.cc` | TOC reading/permutation logic |
| `lib/jxl/enc_toc.h` | `WriteGroupOffsets()` |
| `lib/jxl/enc_toc.cc` | TOC writing implementation |

## Key Types

### FrameHeader (`frame_header.h:330`)

Core frame-level metadata written to the bitstream. Key fields:

```cpp
struct FrameHeader : public Fields {
    enum Flags {
        kNoise = 1, kPatches = 2, kSplines = 16,
        kUseDcFrame = 32, kSkipAdaptiveDCSmoothing = 128,
    };

    FrameEncoding encoding;           // kVarDCT or kModular
    FrameType frame_type;             // kRegularFrame, kDCFrame, kReferenceOnly, kSkipProgressive
    uint64_t flags;
    ColorTransform color_transform;   // kXYB, kNone, kYCbCr
    YCbCrChromaSubsampling chroma_subsampling;
    uint32_t group_size_shift;        // only kModular; shifts base 128-pixel group dim
    uint32_t x_qm_scale;             // chromacity quantization multiplier (X channel)
    uint32_t b_qm_scale;             // chromacity quantization multiplier (B channel)
    Passes passes;                    // progressive pass definitions
    bool custom_size_or_origin;
    FrameSize frame_size;
    uint32_t upsampling;
    std::vector<uint32_t> extra_channel_upsampling;
    FrameOrigin frame_origin;
    BlendingInfo blending_info;
    std::vector<BlendingInfo> extra_channel_blending_info;
    AnimationFrame animation_frame;
    bool is_last;
    uint32_t save_as_reference;       // 0-3
    bool save_before_color_transform;
    uint32_t dc_level;                // 1-4 if kDCFrame
    LoopFilter loop_filter;
};
```

### Passes (`frame_header.h:262`)

Describes how AC coefficients are split across progressive passes:

```cpp
struct Passes : public Fields {
    uint32_t num_passes;      // <= kMaxNumPasses (11)
    uint32_t num_downsample;
    uint32_t downsample[kMaxNumPasses];  // downsampling factor per bracket
    uint32_t last_pass[kMaxNumPasses];   // last pass index per bracket
    uint32_t shift[kMaxNumPasses];       // bit-shift per pass (0 for last)
};
```

### FrameDimensions (`frame_dimensions.h:33`)

All dimension computations derive from `FrameHeader::ToFrameDimensions()`:

```cpp
struct FrameDimensions {
    size_t xsize, ysize;              // post-upsampling-divide pixel size
    size_t xsize_blocks, ysize_blocks; // in 8x8 blocks
    size_t xsize_padded, ysize_padded; // padded to block multiple (VarDCT only)
    size_t xsize_groups, ysize_groups; // in groups (group_dim pixels each)
    size_t xsize_dc_groups, ysize_dc_groups; // in DC groups (group_dim * 8 pixels)
    size_t num_groups, num_dc_groups;
    size_t group_dim;                  // (128 >> 1) << group_size_shift pixels
    size_t dc_group_dim;               // group_dim * kBlockDim
};
```

Key constants:
- `kBlockDim = 8` (pixels per block side)
- `kGroupDim = 256` (pixels per default group side)
- `kGroupDimInBlocks = 32` (blocks per group side)
- Default `group_dim` for VarDCT = 256, for modular varies by `group_size_shift`.

### PassesEncoderState (`enc_cache.h:35`)

Central encoder state, created fresh for each frame:

```cpp
struct PassesEncoderState {
    PassesSharedState shared;            // shared enc/dec state
    bool streaming_mode = false;         // streaming or one-shot
    bool initialize_global_state = true; // write global sections?
    size_t dc_group_index = 0;           // current DC group (streaming)

    std::vector<std::unique_ptr<ACImage>> coeffs; // per-pass AC coefficients
    std::vector<std::unique_ptr<BitWriter>> special_frames; // DC frames
    ProgressiveSplitter progressive_splitter;
    CompressParams cparams;

    struct PassData {
        std::vector<std::vector<Token>> ac_tokens; // per-group token lists
        std::vector<uint8_t> context_map;
        EntropyEncodingData codes;                 // ANS histograms
    };
    std::vector<PassData> passes;          // one PassData per progressive pass
    std::vector<size_t> histogram_idx;     // per-group histogram selection

    uint32_t used_acs = 0;                 // bitmask of AC strategy types seen
    std::vector<uint32_t> used_orders;     // per-pass non-default coeff orders
    float x_qm_multiplier = 1.0f;
    float b_qm_multiplier = 1.0f;
};
```

### PassesSharedState (`passes_state.h:48`)

State shared between encoder and decoder:

```cpp
struct PassesSharedState {
    const CodecMetadata* metadata;
    FrameDimensions frame_dim;
    AcStrategyImage ac_strategy;         // per-block AC strategy map
    DequantMatrices matrices;            // dequantization matrices
    Quantizer quantizer{matrices};       // global quantizer
    ImageI raw_quant_field;              // per-block quant level
    ImageB epf_sharpness;               // per-block EPF sharpness
    ColorCorrelationMap cmap;            // chroma-from-luma correlation
    ImageFeatures image_features;        // noise, patches, splines
    size_t coeff_order_size = 0;
    std::vector<coeff_order_t> coeff_orders; // per-pass coefficient scan orders
    ImageB quant_dc;
    Image3F dc_storage;                  // DC coefficients
    BlockCtxMap block_ctx_map;           // entropy context mapping
    size_t num_histograms = 0;           // number of histogram groups
};
```

### PassDefinition / ProgressiveMode (`enc_progressive_split.h:27`)

```cpp
struct PassDefinition {
    size_t num_coefficients;  // 1-8, side of coeff square to keep per 8x8 block
    size_t shift;             // right-shift applied to values (lossy rounding)
    size_t suitable_for_downsampling_of_at_least; // decoder can stop here
};

struct ProgressiveMode {
    size_t num_passes = 1;
    PassDefinition passes[kMaxNumPasses];
};
```

## EncodeFrame Flow (Step by Step)

The public entry point is:

```cpp
Status EncodeFrame(JxlMemoryManager*, const CompressParams& cparams_orig,
                   const FrameInfo&, const CodecMetadata*,
                   JxlEncoderChunkedFrameAdapter& frame_data,
                   const JxlCmsInterface&, ThreadPool*,
                   JxlEncoderOutputProcessorWrapper*, AuxOut*);
```

### Step 0: Speed Tier Adjustments and Parameter Validation

At `enc_frame.cc:2397`:

1. **TectonicPlate handling**: If `speed_tier == kTectonicPlate` and lossy, downgrade to Glacier. If lossless, run a brute-force search over ~20+ parameter combinations (palette colors, group sizes, predictors, WP modes) in parallel, selecting the smallest output. Two phases: first a palette probe (two variants), then 20+ full variants selected based on which palette configuration won.

2. **Lightning handling**: Downgrade to Thunder (Lightning handled externally).

3. **ParamsPostInit**: Clamp `butteraugli_distance >= kMinButteraugliDistance`, auto-select 2x resampling at distance >= 10, sync `ec_resampling`.

4. **Validation**: Non-negative distance, progressive DC limits, JPEG-specific overrides (disable gaborish, EPF, force VarDCT), empty image check, XYB metadata consistency.

### Step 1: Streaming vs One-Shot Decision

At `enc_frame.cc:2532`:

```cpp
if (CanDoStreamingEncoding(cparams, frame_info, *metadata, frame_data)) {
    return EncodeFrameStreaming(...);
} else {
    return EncodeFrameOneShot(...);
}
```

**Streaming mode** requires all of:
- `cparams.buffering != 0`
- Speed tier >= Tortoise (with distance-dependent relaxation)
- Image > 2048x2048 on both axes
- Not JPEG transcoding
- No forced noise or patches
- No progressive DC, no progressive passes
- No resampling, no lossy palette, no max-error mode
- Modular part is lossless or no extra channels/modular mode
- Correct color transform (XYB for VarDCT, None for modular)

### Step 2: EncodeFrameOneShot

At `enc_frame.cc:2152`:

```
1. Create PassesEncoderState
2. SetProgressiveMode (configure pass definitions)
3. Create FrameHeader via MakeFrameHeader()
4. Create ModularFrameEncoder
5. ComputeEncodingData (THE BIG STEP - computes all encoding data)
6. Write frame header
7. PermuteGroups (center-first ordering if enabled)
8. WriteGroupOffsets (TOC)
9. AppendByteAligned(group_codes) -- all group bitstreams
10. Output final frame bytes
```

### Step 3: MakeFrameHeader

At `enc_frame.cc:319`. Sets up:

- Frame type, is_last, name, origin
- Progressive passes via `progressive_splitter.InitPasses()`
- Encoding mode (VarDCT vs Modular, forced VarDCT for JPEG)
- Group size shift (modular only, varies by image size and speed tier)
- Color transform, chroma subsampling (from JPEG data if transcoding)
- Frame flags: noise (distance >= kMinButteraugliForNoise), DC frame, patches
- Loop filter: Gaborish (Hare or slower, VarDCT, distance > 0.5), EPF iterations (0-3 based on distance thresholds [0.7, 1.5, 4.0])
- DC level, upsampling, blending info, animation frame

### Step 4: ComputeEncodingData

At `enc_frame.cc:1470`. The central computation function. Handles both streaming and one-shot modes:

1. **Set frame dimensions** (`shared.frame_dim.Set()` for streaming, `ToFrameDimensions()` for one-shot)

2. **Allocate shared images**: `ac_strategy`, `raw_quant_field`, `epf_sharpness`, `cmap`, `coeff_orders`, `quant_dc`, `dc_storage`

3. **Copy input pixels**: `CopyColorChannels()` and `CopyExtraChannels()` from chunked input source. In streaming mode, adds border pixels (`max_border = kBlockDim`) for Gaborish/AQ context.

4. **Color space conversion**: If XYB and needs transform, call `ToXYB()`. Optionally store linear-light copy for VarDCT at Kitten speed or slower.

5. **Invisible pixel simplification**: If alpha present, unassociated, not lossless, replace alpha=0 pixels with weighted averages of neighbors (better compression, no visual impact).

6. **Pad to block multiple**: `PadImageToBlockMultipleInPlace()`

7. **Chromacity adjustments** (first DC group in streaming, always in one-shot): Distance-based `x_qm_scale` (3-6), pixel-based analysis of X and B channel gradients.

8. **Noise parameter computation**: Photon noise model, manual noise, or automatic noise estimation (for VarDCT, distance >= kMinButteraugliForNoise).

9. **Downsampling**: Color channels downsampled if resampling > 1 (sharper 2x for VarDCT, generic for others). Extra channels downsampled separately.

10. **VarDCT path** (if `encoding == kVarDCT`):
    - Allocate per-pass AC token storage
    - If JPEG transcoding: `ComputeJPEGTranscodingData()`
    - Otherwise: `ComputeVarDCTEncodingData()` (heuristics + coefficient computation)
    - `ComputeAllCoeffOrders()` -- determine non-default coefficient scan orders
    - Set `num_histograms = 1`, allocate `histogram_idx`
    - `TokenizeAllCoefficients()` -- convert quantized coefficients to entropy tokens

11. **Modular path** (if modular_mode or extra channels present):
    - `enc_modular.ComputeEncodingData()` -- modular encoding of color and/or extra channels

12. **Tree and token computation** (one-shot only, conditional on speed/settings):
    - `enc_modular.ComputeTree()` + `enc_modular.ComputeTokens()`

13. **Update flags**: patches, splines based on actual presence

14. **EncodeGroups()** -- encode all sections to BitWriter objects

### Step 5: VarDCT Encoding Sub-pipeline

#### ComputeVarDCTEncodingData (`enc_frame.cc:1107`)

```cpp
Status ComputeVarDCTEncodingData(const FrameHeader&, const Image3F* linear,
                                  Image3F* opsin, const Rect&,
                                  const JxlCmsInterface&, ThreadPool*,
                                  ModularFrameEncoder*, PassesEncoderState*,
                                  AuxOut*);
```

1. Save pre-Gaborish opsin as `orig_opsin` (for AR heuristics)
2. **LossyFrameHeuristics**: AC strategy selection, quantization field, chroma-from-luma, Gaborish filtering
3. **InitializePassesEncoder**: Forward DCT, quantization, coefficient splitting
4. **ComputeARHeuristics**: Adaptive restoration (EPF sharpness) using original opsin
5. **ComputeACMetadata**: DC quantization and AC metadata for modular DC encoding

#### ComputeJPEGTranscodingData (`enc_frame.cc:764`)

For JPEG-to-JXL lossless transcoding:
- Force DCT8x8, no CfL, no EPF sharpness
- Convert JPEG quantization tables to JXL Quantizer
- Transpose coefficients (JPEG stores row-major, JXL column-major)
- Compute chroma-from-luma correlation (for 4:4:4 with CfL enabled)
- Split coefficients across progressive passes via `SplitACCoefficients()`

### Step 6: ComputeAllCoeffOrders (`enc_frame.cc:1140`)

Determines per-pass non-default coefficient scan orders:

```cpp
auto used_orders_info = ComputeUsedOrders(speed_tier, ac_strategy, raw_quant_field);
for (size_t i = 0; i < num_passes; i++) {
    ComputeCoeffOrder(speed_tier, coeffs[i], ac_strategy, frame_dim,
                      used_orders[i], used_acs, ...);
}
```

### Step 7: TokenizeAllCoefficients (`enc_frame.cc:1172`)

Parallelized over groups. Per group, per pass:

```cpp
TokenizeCoefficients(&coeff_orders[pass * coeff_order_size], rect,
                     ac_rows, ac_strategy, chroma_subsampling,
                     &num_nzeroes, &ac_tokens[group_index],
                     quant_dc, raw_quant_field, block_ctx_map);
```

Uses `EncCache` per thread with `Image3I num_nzeroes` working space.

## Group Encoding

### TOC Section Layout

For a frame with `num_groups > 1` or `num_passes > 1`, the TOC has these sections in order:

| Index | Section |
|-------|---------|
| 0 | DC Global (patches, splines, noise, DC dequant matrices, quantizer params, block context map, CfL DC, modular global info, modular global stream) |
| 1 .. num_dc_groups | DC Groups (VarDCT DC precision + stream, modular DC stream, AC metadata stream) |
| num_dc_groups + 1 | AC Global (dequant matrices, num_histograms, per-pass coeff orders, per-pass ANS histograms) |
| num_dc_groups + 2 .. end | AC Groups: pass 0 group 0, pass 0 group 1, ..., pass 1 group 0, ... |

The index formula for AC groups:
```cpp
size_t AcGroupIndex(pass, group, num_groups, num_dc_groups) {
    return 2 + num_dc_groups + pass * num_groups + group;
}
```

**Small image optimization**: If `num_groups == 1 && num_passes == 1`, all sections are merged into a single TOC entry.

### EncodeGroups (`enc_frame.cc:1299`)

```cpp
Status EncodeGroups(const FrameHeader&, PassesEncoderState*,
                    ModularFrameEncoder*, ThreadPool*,
                    std::vector<std::unique_ptr<BitWriter>>* group_codes,
                    AuxOut*);
```

Allocates one `BitWriter` per TOC entry. Three parallel phases:

**Phase 1: DC Global** (index 0, serial):
- Patch dictionary encoding
- Spline encoding
- Noise parameters
- DC dequant matrices
- Global DC info (quantizer params, block context map, CfL DC correlation)
- Modular global info and global stream

**Phase 2: DC Groups** (indices 1..num_dc_groups, parallel via `RunOnPool`):
Per DC group:
- VarDCT DC precision bits + modular DC stream
- Modular DC group stream
- AC metadata size + AC metadata stream (quant field, EPF sharpness, AC strategy)

**Phase 3: AC Global** (index num_dc_groups+1, serial):
- `EncodeGlobalACInfo()`: dequant matrices, num_histograms, per-pass coefficient orders + histograms
- ANS histogram clustering with `BuildAndEncodeHistograms()`

**Phase 4: AC Groups** (all remaining indices, parallel via `RunOnPool`):
Per group, per pass:
- `EncodeGroupTokenizedCoefficients()`: histogram selector bits + `WriteTokens()` for AC
- Modular AC group stream

Each BitWriter is zero-padded to byte boundary after encoding.

### ComputeCoefficients (`enc_group.cc:382`)

Per-group VarDCT coefficient computation (HWY-dispatched). Per block:

1. Forward DCT all 3 channels from opsin pixels
2. Extract DC from lowest frequencies
3. `QuantizeRoundtripYBlockAC()`: quantize Y, dequantize Y for CfL reference
   - `AdjustQuantBlockAC()` at Hare or slower: adapts quantization for flat blocks, high-frequency patterns
   - `QuantizeBlockAC()`: SIMD quantization with dead-zone thresholds
4. Unapply chroma-from-luma correlation (subtract Y contribution from X and B)
5. Quantize X and B channels, extract their DC
6. Split quantized coefficients across passes via `progressive_splitter.SplitACCoefficients()`

### EncodeGroupTokenizedCoefficients (`enc_group.cc:545`)

Per AC group in bitstream:
```cpp
// Write histogram selector (log2(num_histograms) bits)
writer->Write(histo_selector_bits, histogram_idx);
// Write entropy-coded tokens
WriteTokens(ac_tokens[group_idx], codes, context_offset, writer, ...);
```

## Progressive Modes

### Built-in Configurations

**SetProgressiveMode** (`enc_frame.cc:222`) selects from:

**progressive_mode** (3 passes: DC+VLF, +LF, +full AC):
```
Pass 0: num_coefficients=2, shift=0, downsample >= 4
Pass 1: num_coefficients=3, shift=0, downsample >= 2
Pass 2: num_coefficients=8, shift=0, downsample >= 0
```

**qprogressive_mode** (2 passes: quantized AC, full AC):
```
Pass 0: num_coefficients=8, shift=1, downsample >= 2
Pass 1: num_coefficients=8, shift=0, downsample >= 0
```

**custom_progressive_mode**: arbitrary user-defined pass definitions.

**Default**: 1 pass, all coefficients, no shift.

### SplitACCoefficients (`enc_progressive_split.cc:21`)

For each progressive pass, determines which coefficients belong to it:

```
For each pass:
  For each (y,x) in [0..ysize*frame_ncoeffs) x [0..xsize*frame_ncoeffs):
    Skip if already covered by an earlier pass with smaller num_coefficients
    Subtract contribution of previous pass (if previous had shift > 0)
    Apply current pass shift: output[pass][pos] = shift_right_round0(v, shift)
```

The `shift_right_round0` function rounds toward zero (adds correction for negative values before right-shifting).

After a pass with `shift == 0`, all coefficients up to `num_coefficients` are considered fully done.

### Passes Initialization

`ProgressiveSplitter::InitPasses()` populates the `Passes` struct in the frame header:
- Sets `num_passes`, `shift[]` per pass
- Populates `downsample[]` and `last_pass[]` arrays for decoder downsampling hints
- The last pass always has `shift = 0`

## TOC and Bitstream Assembly

### TOC Distribution

```cpp
constexpr U32Enc kTocDist(Bits(10),           // 0..1023 (2 + 10 = 12 bits)
                           BitsOffset(14, 1024),   // 1024..17407 (2 + 14 = 16 bits)
                           BitsOffset(22, 17408),  // 17408..4211711 (2 + 22 = 24 bits)
                           BitsOffset(30, 4211712)); // 4211712+ (2 + 30 = 32 bits)
```

Four size buckets with offsets:
- Bucket 0: sizes 0..1023 (12-bit encoding)
- Bucket 1: sizes 1024..17407 (16-bit encoding)
- Bucket 2: sizes 17408..4211711 (24-bit encoding)
- Bucket 3: sizes 4211712+ (32-bit encoding)

### One-Shot Bitstream Assembly

`EncodeFrameOneShot` at `enc_frame.cc:2152`:

```
[special_frames (DC frames)]     -- byte-aligned prepend
[frame_header]                   -- WriteFrameHeader
[permutation_bit + permutation]  -- WriteGroupOffsets
[TOC entries]                    -- U32 sizes per section
[byte padding]                   -- zero-pad to byte
[group_codes[0]]                 -- DC global section
[group_codes[1..N]]              -- remaining sections, byte-aligned
```

**WriteGroupOffsets** (`enc_toc.cc:23`):
1. Write 1 bit: permutation present?
2. If permutation: `EncodePermutation()` using Lehmer code
3. Zero-pad to byte
4. Write each group's byte-size using `kTocDist`
5. Zero-pad to byte

### Group Permutation (Center-First)

`PermuteGroups` (`enc_frame.cc:1688`): If `cparams.centerfirst`, reorders AC groups by distance from image center (or specified center coordinates). Uses concentric-square ordering with clockwise angular sort within each distance ring. The permutation is:
- DC global and DC groups: identity (unchanged)
- AC groups per pass: sorted by distance from center

### Streaming Bitstream Assembly

`EncodeFrameStreaming` at `enc_frame.cc:2031`. Processes one DC group at a time:

1. **Pre-compute permutation** via `ComputePermutationForStreaming()`:
   - DC global first
   - Then for each DC group (raster order): DC group section, followed by all its AC groups (all passes)
   - AC global last

2. **Compute group data offset**: determines where group data begins in the output, accounting for variable TOC size. The DC global section is padded so that group data starts at a predictable offset.

3. **Seek past header area**: write group data first (forward-only writes)

4. **Per DC group** (sequential):
   - First iteration: encode frame header, write DC global bytes, compute offset
   - `ComputeEncodingData()` for this DC group's region
   - `OutputGroups()`: write DC group + its AC groups sequentially

5. **AC Global**: written last (after all groups, since histograms need all tokens)
   - `OutputAcGlobal()`: writes dequant matrices (1 bit = default), num_histograms = num_dc_groups, coeff orders, cleaned-up histograms

6. **Seek back**: write frame header + TOC + DC global + padding at the reserved space

The streaming TOC padding logic ensures the TOC + DC global occupies exactly the pre-computed size:
```cpp
ComputeGroupDataOffset(frame_header_size, dc_global_size, num_sections,
                       min_dc_global_size, group_data_offset);
// min_dc_global_size >= dc_global_size such that TOC bucket doesn't change
// even with maximum TOC compression variance
padding_size = group_data_offset - actual_offset;
```

### Streaming Permutation Order

```cpp
ComputePermutationForStreaming(xsize, ysize, group_size, num_passes,
                                permutation, dc_group_order);
```

The permutation interleaves DC and AC groups for streaming-friendly order:
```
[DC Global]
For each DC group (y, x) in raster order:
    [DC group]
    For each pass:
        For each AC group in this DC group (raster order within):
            [AC group (pass, group)]
[AC Global]  // last
```

This allows a streaming decoder to process DC groups and their corresponding AC groups incrementally.

## Dependencies

### Upstream (read by encoder pipeline)
- `enc_params.h` -- `CompressParams`, `SpeedTier` enum
- `image_metadata.h` -- `CodecMetadata`, `ImageMetadata`, `ExtraChannelInfo`
- `enc_xyb.h` -- `ToXYB()` color transform
- `enc_heuristics.h` -- `LossyFrameHeuristics()` (AC strategy, quant field selection)
- `enc_adaptive_quantization.h` -- `ComputeARHeuristics()` (EPF sharpness)
- `enc_chroma_from_luma.h` -- CfL computation
- `enc_ac_strategy.h` -- AC strategy selection
- `enc_quant_weights.h` -- Quantization matrix setup
- `enc_modular.h` -- `ModularFrameEncoder` (DC, metadata, extra channels)
- `enc_ans.h` -- `BuildAndEncodeHistograms()`, `WriteTokens()`
- `enc_entropy_coder.h` -- `TokenizeCoefficients()`
- `enc_coeff_order.h` -- `ComputeCoeffOrder()`, `ComputeUsedOrders()`
- `enc_noise.h` -- Noise parameter estimation
- `enc_splines.h` -- Spline encoding
- `enc_patch_dictionary.h` -- Patch encoding

### Downstream (decoder counterparts)
- `toc.h` / `toc.cc` -- TOC reading and permutation decoding
- `frame_header.h` -- Frame header reading
- `passes_state.h` -- Shared decoder state
- `dec_modular.h` -- Modular decoding
- `dec_transforms-inl.h` -- Inverse DCT (used in roundtrip quantization)

### Internal Parallelization Points

1. `ComputeJPEGTranscodingData` -- DC group metadata computation (`RunOnPool`)
2. `TokenizeAllCoefficients` -- per-group tokenization (`RunOnPool`)
3. `EncodeGroups` -- DC group encoding (`RunOnPool`), AC group encoding (`RunOnPool`)
4. `TectonicPlate` brute-force search -- all parameter variants (`RunOnPool`)
5. `ComputeCoefficients` -- called per-group from `InitializePassesEncoder` via pool
6. CfL correlation computation -- per-tile rows (`RunOnPool`)

All pool tasks follow the pattern `RunOnPool(pool, 0, count, init_fn, work_fn, label)` where `init_fn` allocates per-thread scratch space and `work_fn` processes one unit of work.

# Analysis Progress

## Phase 1: Foundations
- [x] 01-base-primitives (status.h, common.h, span.h, bits.h)
- [x] 02-memory-model (image.h, Plane<T>, alignment)
- [x] 03-serialization (fields.h, Visitor pattern, Bundle)
- [x] 04-bit-io (BitReader, BitWriter, PaddedBytes)

## Phase 2: Entropy Coding
- [x] 05-ans-entropy (ans_common/params/enc_ans/dec_ans, alias tables, hybrid integers, encoding/decoding)
- [x] 08-histogram-clustering (enc_cluster, context maps, LZ77, block context)

## Phase 3: Color Science
- [x] 11-xyb-color-space (opsin absorbance, XYB encoding)
- [ ] 12-transfer-functions (sRGB, PQ, HLG, gamma)
- [ ] 13-color-management (CMS interface, ICC profiles)
- [ ] 14-tone-mapping (HDR→SDR, display adaptation)

## Phase 4: Transforms + Quantization
- [x] 15-dct-family (DCT-II/III, sizes, SIMD)
- [x] 16-ac-strategy (block size selection heuristics, COST FUNCTIONS)
- [ ] 17-coefficient-order (scan order optimization)
- [x] 18-quantization (quant matrices, DC/AC quantization, MULTIPLIERS)

## Phase 5: Perceptual Models
- [x] 19-butteraugli (perceptual distance metric)
- [x] 20-adaptive-quantization (AQ masking, DECISION TREES, COST ANALYSIS)
- [x] 21-gaborish (edge enhancement filter)

## Phase 6: Modular Path
- [x] 22-modular-overview (when and why modular, RCT, squeeze, palette, MA trees)

## Phase 7: Features
- [x] 27-features (patches, splines, noise, CfL — combined into one note)

## Phase 8: Encoder Pipeline
- [x] 31-pipeline-overview (full encoder flow, DECISION TREE)
- [ ] 32-frame-setup (FrameHeader, metadata, params)
- [ ] 33-vardct-path (heuristics → DCT → quant → tokens, COST ANALYSIS)
- [ ] 34-group-encoding (256×256 groups, parallel encoding)
- [ ] 35-progressive (DC/AC/QAC progressive modes)
- [ ] 36-bitstream-assembly (TOC, group permutation, output)

## Phase 9: Architecture
- [ ] 37-highway-simd (HWY abstraction, -inl.h pattern)
- [ ] 38-render-pipeline (decoder pipeline reference)
- [ ] 39-threading (ThreadPool, RunOnPool, group parallelism)

## Chapters Written
- [x] foundations/base-primitives.md
- [x] foundations/memory-model.md
- [x] foundations/serialization.md
- [x] foundations/bit-io.md
- [x] entropy/ans-theory.md
- [x] entropy/ans-implementation.md
- [x] entropy/hybrid-integers.md
- [x] entropy/histogram-clustering.md
- [x] entropy/lz77.md
- [x] entropy/context-modeling.md

## Remaining Analyses Needed
- 12-transfer-functions, 13-color-management, 14-tone-mapping
- 17-coefficient-order
- 32-frame-setup, 33-vardct-path, 34-group-encoding, 35-progressive, 36-bitstream-assembly
- 37-highway-simd, 38-render-pipeline, 39-threading

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
- [x] 12-color-management (transfer functions, CMS interface, ICC profiles, tone mapping)

## Phase 4: Transforms + Quantization
- [x] 15-dct-family (DCT-II/III, sizes, SIMD)
- [x] 16-ac-strategy (block size selection heuristics, COST FUNCTIONS)
- [x] 17-coefficient-order (scan order optimization, Lehmer code)
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
- [x] 32-encoder-pipeline (frame setup, group encoding, progressive, bitstream assembly)

## Phase 9: Architecture
- [x] 37-architecture (Highway SIMD, render pipeline, threading model)

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
- [x] color/xyb-color-space.md
- [x] color/transfer-functions.md
- [x] color/color-management.md
- [x] color/tone-mapping.md
- [x] transforms/dct-family.md
- [x] transforms/ac-strategy.md
- [x] transforms/coefficient-order.md
- [x] transforms/quantization.md
- [x] perceptual/butteraugli.md
- [x] perceptual/adaptive-quantization.md
- [x] perceptual/gaborish.md
- [x] modular/modular-overview.md
- [x] modular/rct.md
- [x] modular/squeeze.md
- [x] modular/palette.md
- [x] modular/prediction-trees.md
- [x] features/chroma-from-luma.md
- [x] features/noise.md
- [x] features/patches.md
- [x] features/splines.md
- [x] encoder/pipeline-overview.md
- [x] encoder/frame-setup.md
- [x] encoder/vardct-path.md
- [x] encoder/group-encoding.md
- [x] encoder/progressive.md
- [x] encoder/bitstream-assembly.md
- [x] architecture/highway-simd.md
- [x] architecture/render-pipeline.md
- [x] architecture/threading.md

## Status: COMPLETE
All 15 analysis notes and 39 chapters written.

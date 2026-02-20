# Analysis Progress

## Phase 1: Foundations
- [ ] 01-base-primitives (status.h, common.h, span.h, bits.h)
- [ ] 02-memory-model (image.h, Plane<T>, alignment)
- [ ] 03-serialization (fields.h, Visitor pattern, Bundle)
- [ ] 04-bit-io (BitReader, BitWriter, PaddedBytes)

## Phase 2: Entropy Coding
- [ ] 05-ans-common (ans_common.h, alias tables)
- [ ] 06-ans-encoder (enc_ans.cc, symbol writing)
- [ ] 07-hybrid-integers (HybridUintConfig, split-exponent)
- [ ] 08-histogram-clustering (enc_cluster, distance metrics)
- [ ] 09-lz77 (enc_lz77, back-references)
- [ ] 10-context-modeling (context maps, block context)

## Phase 3: Color Science
- [ ] 11-xyb-color-space (opsin absorbance, XYB encoding)
- [ ] 12-transfer-functions (sRGB, PQ, HLG, gamma)
- [ ] 13-color-management (CMS interface, ICC profiles)
- [ ] 14-tone-mapping (HDR→SDR, display adaptation)

## Phase 4: Transforms + Quantization
- [ ] 15-dct-family (DCT-II/III, sizes, SIMD)
- [ ] 16-ac-strategy (block size selection heuristics, COST FUNCTIONS)
- [ ] 17-coefficient-order (scan order optimization)
- [ ] 18-quantization (quant matrices, DC/AC quantization, MULTIPLIERS)

## Phase 5: Perceptual Models
- [ ] 19-butteraugli (perceptual distance metric)
- [ ] 20-adaptive-quantization (AQ masking, DECISION TREES, COST ANALYSIS)
- [ ] 21-gaborish (edge enhancement filter)

## Phase 6: Modular Path
- [ ] 22-modular-overview (when and why modular)
- [ ] 23-rct (Reversible Color Transform)
- [ ] 24-squeeze (Haar-like decorrelation)
- [ ] 25-palette (indexed color)
- [ ] 26-prediction-trees (MA trees, context prediction)

## Phase 7: Features
- [ ] 27-patches (patch dictionary)
- [ ] 28-splines (parametric curves)
- [ ] 29-noise (photon/film noise synthesis)
- [ ] 30-chroma-from-luma (CfL for JPEG transcoding)

## Phase 8: Encoder Pipeline
- [ ] 31-pipeline-overview (full encoder flow, DECISION TREE)
- [ ] 32-frame-setup (FrameHeader, metadata, params)
- [ ] 33-vardct-path (heuristics → DCT → quant → tokens, COST ANALYSIS)
- [ ] 34-group-encoding (256×256 groups, parallel encoding)
- [ ] 35-progressive (DC/AC/QAC progressive modes)
- [ ] 36-bitstream-assembly (TOC, group permutation, output)

## Phase 9: Architecture
- [ ] 37-highway-simd (HWY abstraction, -inl.h pattern)
- [ ] 38-render-pipeline (decoder pipeline reference)
- [ ] 39-threading (ThreadPool, RunOnPool, group parallelism)

# Architecture: SIMD & Threading

## Source Files

- `/home/lilith/work/jxl-efforts/libjxl/lib/jxl/base/fast_math-inl.h` - SIMD math approximations (log2, pow2, cos, erf, cbrt)
- `/home/lilith/work/jxl-efforts/libjxl/lib/jxl/base/rational_polynomial-inl.h` - SIMD rational polynomial evaluation (Horner scheme)
- `/home/lilith/work/jxl-efforts/libjxl/lib/jxl/simd_util.h` / `simd_util.cc` - MaxVectorSize query and SIMD MaxValue reduction
- `/home/lilith/work/jxl-efforts/libjxl/lib/jxl/enc_transforms-inl.h` - Forward DCT/IDCT transforms (all AC strategy types)
- `/home/lilith/work/jxl-efforts/libjxl/lib/jxl/render_pipeline/render_pipeline.h` - Pipeline interface and Builder
- `/home/lilith/work/jxl-efforts/libjxl/lib/jxl/render_pipeline/render_pipeline_stage.h` - Stage base class
- `/home/lilith/work/jxl-efforts/libjxl/lib/jxl/render_pipeline/simple_render_pipeline.cc` - Non-memory-efficient pipeline implementation
- `/home/lilith/work/jxl-efforts/libjxl/lib/jxl/render_pipeline/low_memory_render_pipeline.h` - Production pipeline with border sharing
- `/home/lilith/work/jxl-efforts/libjxl/lib/jxl/render_pipeline/stage_write.cc` - Output stage (SIMD pixel format conversion)
- `/home/lilith/work/jxl-efforts/libjxl/lib/jxl/render_pipeline/stage_xyb.cc` - XYB-to-RGB color space conversion stage
- `/home/lilith/work/jxl-efforts/libjxl/lib/jxl/base/data_parallel.h` - ThreadPool wrapper and RunOnPool
- `/home/lilith/work/jxl-efforts/libjxl/lib/include/jxl/parallel_runner.h` - C API for pluggable thread runners
- `/home/lilith/work/jxl-efforts/libjxl/lib/threads/thread_parallel_runner_internal.h` - std::thread-based runner implementation

## Highway SIMD Pattern

libjxl uses Google's [Highway](https://github.com/google/highway) library for portable SIMD. Highway provides a C++ API where SIMD operations are expressed as function calls on abstract vector types, and the library generates optimized code for multiple instruction set targets (SSE4, AVX2, AVX-512, NEON, WASM SIMD, etc.) from a single source.

The core pattern is:
1. Write SIMD code once using Highway's `HWY_FULL(float)`, `LoadU`, `StoreU`, `Mul`, `MulAdd`, etc.
2. The preprocessor re-includes the same source file once per target architecture.
3. At runtime, a CPU feature check selects the best available implementation.

Highway vectors are not a fixed width. `HWY_FULL(float)` creates a descriptor for the widest native float vector on the current target, which could be 128-bit (SSE/NEON), 256-bit (AVX2), or 512-bit (AVX-512). Code loops in increments of `Lanes(d)`, which returns the number of elements per vector. This means the same code automatically adapts from 4-wide to 16-wide processing.

## -inl.h Mechanism

The `-inl.h` suffix denotes "inline headers" that are designed to be compiled multiple times -- once per SIMD target. This is the central build pattern throughout libjxl.

### The include guard / re-include pattern

Standard include guards prevent a header from being processed twice. The `-inl.h` files use a toggling guard that *allows* re-inclusion. From `fast_math-inl.h` (line 10-15):

```cpp
#if defined(LIB_JXL_BASE_FAST_MATH_INL_H_) == defined(HWY_TARGET_TOGGLE)
#ifdef LIB_JXL_BASE_FAST_MATH_INL_H_
#undef LIB_JXL_BASE_FAST_MATH_INL_H_
#else
#define LIB_JXL_BASE_FAST_MATH_INL_H_
#endif
```

`HWY_TARGET_TOGGLE` is defined/undefined by Highway's `foreach_target.h` on alternating passes. Each time the `.cc` file is re-included, the toggle state flips, the guard flips, and the body is processed again. The result: the code inside is compiled once per target (e.g., once for SSE4, once for AVX2, once for AVX-512).

### Namespace isolation

Each re-inclusion places code into a different namespace:

```cpp
HWY_BEFORE_NAMESPACE();
namespace jxl {
namespace HWY_NAMESPACE {  // expands to e.g. N_SSE4, N_AVX2, N_AVX3
  // ... SIMD code ...
}  // namespace HWY_NAMESPACE
}  // namespace jxl
HWY_AFTER_NAMESPACE();
```

`HWY_NAMESPACE` is a macro that expands to a target-specific name like `N_SSE4`, `N_AVX2`, `N_AVX3`, or `N_AVX3_ZEN4`. This prevents ODR violations -- each target's functions live in their own namespace.

### ADL workaround

Highway SIMD functions live in `hwy::HWY_NAMESPACE`, and template-based operations are not found via ADL (argument-dependent lookup). Every `-inl.h` file begins with `using` declarations to import the needed operations:

```cpp
using hwy::HWY_NAMESPACE::Abs;
using hwy::HWY_NAMESPACE::Add;
using hwy::HWY_NAMESPACE::Mul;
using hwy::HWY_NAMESPACE::MulAdd;
// ... etc
```

### The .cc file structure

A `.cc` file that uses Highway SIMD follows a rigid template. Taking `stage_xyb.cc` as a representative example:

```cpp
// (1) Regular includes that don't depend on SIMD target
#include "lib/jxl/render_pipeline/stage_xyb.h"
#include <cstddef>
// ...

// (2) Tell Highway which file to re-include
#undef HWY_TARGET_INCLUDE
#define HWY_TARGET_INCLUDE "lib/jxl/render_pipeline/stage_xyb.cc"
#include <hwy/foreach_target.h>     // triggers multiple re-includes of this file
#include <hwy/highway.h>

// (3) Includes that need to be re-processed per target (the -inl.h files)
#include "lib/jxl/dec_xyb-inl.h"

// (4) SIMD code in target-specific namespace
HWY_BEFORE_NAMESPACE();
namespace jxl {
namespace HWY_NAMESPACE {

class XYBStage : public RenderPipelineStage { /* SIMD ProcessRow */ };

std::unique_ptr<RenderPipelineStage> GetXYBStage(
    const OutputEncodingInfo& output_encoding_info) {
  return jxl::make_unique<XYBStage>(output_encoding_info);
}

}  // namespace HWY_NAMESPACE
}  // namespace jxl
HWY_AFTER_NAMESPACE();

// (5) One-time dispatch registration (compiled only once)
#if HWY_ONCE
namespace jxl {
HWY_EXPORT(GetXYBStage);
std::unique_ptr<RenderPipelineStage> GetXYBStage(
    const OutputEncodingInfo& output_encoding_info) {
  return HWY_DYNAMIC_DISPATCH(GetXYBStage)(output_encoding_info);
}
}  // namespace jxl
#endif
```

`foreach_target.h` sets `HWY_TARGET` to each supported target in turn, then re-includes the file specified by `HWY_TARGET_INCLUDE`. The sections before `foreach_target.h` are only processed once. Sections 3-4 are processed once per target. Section 5 (inside `#if HWY_ONCE`) is processed only on the final pass.

## HWY_DYNAMIC_DISPATCH

`HWY_EXPORT(FunctionName)` creates a function pointer table with one entry per target. `HWY_DYNAMIC_DISPATCH(FunctionName)` returns a pointer to the best available implementation at runtime.

From `simd_util.cc` (lines 54-69):

```cpp
#if HWY_ONCE
namespace jxl {

HWY_EXPORT(MaxVectorSize);
HWY_EXPORT(MaxValue);

size_t MaxVectorSize() {
  return HWY_DYNAMIC_DISPATCH(MaxVectorSize)();
}

uint32_t MaxValue(uint32_t* JXL_RESTRICT data, size_t len) {
  return HWY_DYNAMIC_DISPATCH(MaxValue)(data, len);
}

}  // namespace jxl
#endif
```

There is also `HWY_STATIC_DISPATCH`, used for scalar convenience wrappers. In `fast_math-inl.h` (lines 223-238), the scalar `FastLog2f(float)` calls `HWY_STATIC_DISPATCH(FastLog2f)` -- this bypasses runtime dispatch and calls the compile-time target's version, which for a scalar wrapper with `HWY_CAPPED(float, 1)` operates on a single lane.

### Render pipeline stage pattern

Render pipeline stages use a variation where the entire class is defined inside `HWY_NAMESPACE`. The factory function (e.g., `GetXYBStage`) is what gets exported via `HWY_EXPORT`. The returned `std::unique_ptr<RenderPipelineStage>` points to a target-specific class instance whose `ProcessRow` virtual method contains SIMD code compiled for that specific target. This lets the entire class body -- including all its SIMD loops -- be specialized per target without exposing the target-specific class name outside the namespace.

## Key SIMD Utilities (fast_math-inl.h)

`fast_math-inl.h` provides vectorized approximations of transcendental functions used throughout the encoder and decoder:

**FastLog2f** (line 48, L1 error ~3.9e-6): Base-2 logarithm via range reduction to [-1/3, 1/3] followed by a (2,2) rational polynomial approximation. Extracts the IEEE 754 exponent via integer bit manipulation (`ShiftRight<23>`) and evaluates the mantissa polynomial with `EvalRationalPolynomial` from `rational_polynomial-inl.h`. Used pervasively in entropy coding cost estimation.

**FastPow2f** (line 72, max relative error ~3e-7): Base-2 exponentiation. Separates integer and fractional parts; the integer part becomes the IEEE exponent directly via `ShiftLeft<23>`, and the fractional part is evaluated with a (3,3) rational polynomial (expressed as Horner form, not via `EvalRationalPolynomial`).

**FastPowf** (line 90): Computes `base^exponent` as `2^(log2(base) * exponent)`, composing FastLog2f and FastPow2f. Max relative error ~3e-5.

**FastCosf** (line 97, L1 error ~7e-5): Range reduction to [0, pi/2], Taylor-like approximation scaled by 2^0.75, then two angle-duplication steps to recover the full range. Sign correction via bit manipulation.

**FastErff** (line 129, L1 error ~7e-4): Error function approximation using the formula `1 - 1/((ax*a + b)*x + c)*x + d)*x + 1)^4`. Used for spline rendering.

**CubeRootAndAdd** (line 179): Cube root via initial exponent estimate (multiply exponent by -1/3 in integer domain) followed by 3 Newton-Raphson iterations plus a final refinement. Returns `cbrt(x) + add`, fusing the addition to avoid a separate pass. Based on Agner Fog's vectorclass, used in color space transforms (the cube root appears in L*a*b* and related perceptual spaces).

**EvalRationalPolynomial** (from `rational_polynomial-inl.h`): Evaluates P(x)/Q(x) where P and Q are polynomials stored as aligned coefficient arrays with `HWY_REP4` replication (for `LoadDup128`). Uses Horner's scheme (more efficient than Clenshaw for this use case). The division uses a Newton-Raphson reciprocal approximation for float types, though the current code defaults to hardware division (`Div`) on modern architectures like Skylake-X where `vdivps` throughput is adequate.

## Render Pipeline Architecture

The render pipeline is libjxl's processing framework for the decoder. It chains a sequence of stages that transform decoded coefficient data into final pixel output.

### Pipeline structure

A `RenderPipeline` is built via `RenderPipeline::Builder`:
1. Stages are added in order via `AddStage`.
2. `Finalize()` computes cumulative padding requirements and channel shift tables, then creates either `SimpleRenderPipeline` or `LowMemoryRenderPipeline`.
3. `PrepareForThreads(num, use_group_ids)` allocates per-thread or per-group working buffers.

The pipeline tracks `channel_shifts_` -- at each stage, each channel may have a different resolution (e.g., chroma subsampling means chroma channels are half-size at early stages). Stages with `kInOut` channel mode produce output at a potentially different resolution (controlled by `shift_x` / `shift_y` in `Settings`).

### Stage types

From `RenderPipelineChannelMode`:
- **kIgnored**: Stage does not touch this channel.
- **kInPlace**: Stage modifies the channel buffer in-place (no shift, no extra border).
- **kInOut**: Stage reads input with padding (`border_x`, `border_y`) and writes to a separate output buffer, potentially at a different resolution.
- **kInput**: Stage only reads. This is the terminal mode -- stages that produce observable output (writing pixels to the output buffer) use this.

### Key stages

The stages form a pipeline from internal representation to output pixels:
- **XYBStage** (`stage_xyb.cc`): Converts from XYB color space to linear RGB. Operates kInPlace on channels 0-2. The `ProcessRow` loop uses `HWY_FULL(float)`, processing `Lanes(d)` pixels per iteration with `LoadU`/`StoreU` and fused multiply-add operations.
- **GaborishStage** (`stage_gaborish.cc`): Edge-preserving smoothing filter.
- **UpsamplingStage** (`stage_upsampling.cc`): 2x/4x/8x upsampling (kInOut with shift).
- **NoiseStage** (`stage_noise.cc`): Adds film grain noise.
- **FromLinearStage** / **ToLinearStage**: Transfer function application.
- **ToneMappingStage**: HDR tone mapping.
- **WriteToOutputStage** (`stage_write.cc`): Terminal stage (kInput). Converts float planar data to interleaved integer pixels with ordered dithering. Uses `MakeUnsigned<T>` with blue noise dither for 8-bit output, `StoreInterleaved2/3/4` for channel interleaving. Per-thread temporary buffers avoid contention.

### Two pipeline implementations

**SimpleRenderPipeline**: Allocates full-frame buffers for each channel. Processes all groups into these buffers, then runs stages sequentially over the full frame. Simple but uses O(width * height) memory. Primarily used for testing and as a reference implementation.

**LowMemoryRenderPipeline**: The production implementation. Processes data group-by-group, only allocating buffers sized for one group plus borders. Key features:
- `group_data_` is indexed by `[thread][channel]` or `[group][channel]` depending on `use_group_ids_`.
- Borders between adjacent groups are saved to `borders_horizontal_` and `borders_vertical_` buffers, then loaded back when a neighboring group is processed. This is managed by `GroupBorderAssigner`.
- `stage_data_` provides intermediate row buffers indexed by `[thread][channel][stage]`.
- Processing is row-by-row within each group through `RenderRect`.

### Pipeline input flow

The caller provides input via `RenderPipelineInput`:
```cpp
RenderPipelineInput input = pipeline->GetInputBuffers(group_id, thread_id);
// ... fill input.GetBuffer(channel) with decoded data ...
input.Done();  // triggers ProcessBuffers
```

`Done()` calls `InputReady()` which increments `group_completed_passes_[group_id]` and triggers `ProcessBuffers` for that group. This design allows concurrent filling of different groups from different threads.

## Threading Model

### JxlParallelRunner interface

libjxl defines a pluggable threading interface in `parallel_runner.h`. The core callback type is:

```c
typedef JxlParallelRetCode (*JxlParallelRunner)(
    void* runner_opaque, void* jpegxl_opaque,
    JxlParallelRunInit init,
    JxlParallelRunFunction func,
    uint32_t start_range, uint32_t end_range);
```

The runner must:
1. Call `init(jpegxl_opaque, num_threads)` once on the calling thread.
2. Call `func(jpegxl_opaque, value, thread_id)` for each value in `[start_range, end_range)`, possibly from different threads.

This design separates threading policy from the codec entirely. Callers can provide their own runner or use `JxlThreadParallelRunner` from `libjxl_threads`.

### ThreadPool wrapper

`data_parallel.h` provides `ThreadPool`, a thin C++ wrapper:

```cpp
class ThreadPool {
  // Runs init_func(num_threads) then data_func(task, thread) for [begin, end)
  template <class InitFunc, class DataFunc>
  Status Run(uint32_t begin, uint32_t end,
             const InitFunc& init_func, const DataFunc& data_func,
             const char* caller);
};
```

If no runner was provided (null), it falls back to sequential execution on the calling thread. The `RunCallState` inner class bridges C++ lambdas to the C callback interface, with an `atomic<uint32_t> has_error_` for thread-safe error propagation.

### RunOnPool pattern

The free function `RunOnPool` is the dominant parallelism idiom throughout libjxl:

```cpp
template <class InitFunc, class DataFunc>
Status RunOnPool(ThreadPool* pool, uint32_t begin, uint32_t end,
                 const InitFunc& init_func, const DataFunc& data_func,
                 const char* caller);
```

An overload accepts `ThreadPool::NoInit` when no per-thread initialization is needed.

### How groups are parallelized

The image is divided into groups of `group_dim x group_dim` pixels (typically 256x256 at default settings, computed from `kGroupDim` and `group_size_shift`). From `FrameDimensions`:

```cpp
group_dim = (kGroupDim >> 1) << group_size_shift;  // typically 256
xsize_groups = DivCeil(xsize, group_dim);
ysize_groups = DivCeil(ysize, group_dim);
num_groups = xsize_groups * ysize_groups;
num_dc_groups = xsize_dc_groups * ysize_dc_groups;
```

Each group gets a linear index `group_id = gy * xsize_groups + gx`. The encoder and decoder parallelize at the group level:

**Encoder** (`enc_frame.cc`):
```cpp
// Process DC groups in parallel
RunOnPool(pool, 0, shared.frame_dim.num_dc_groups,
          ThreadPool::NoInit, compute_dc_coeffs, "Compute DC coeffs");

// Tokenize AC groups in parallel
RunOnPool(pool, 0, shared.frame_dim.num_groups,
          tokenize_group_init, tokenize_group, "TokenizeGroup");

// Encode groups in parallel
RunOnPool(pool, 0, num_groups, resize_aux_outs,
          process_group, "EncodeGroupCoefficients");
```

**Decoder** (`dec_frame.cc`):
```cpp
// Decode DC groups in parallel
RunOnPool(pool_, 0, dc_group_sec.size(),
          ThreadPool::NoInit, process_section, "DecodeDCGroup");

// Decode AC groups in parallel
RunOnPool(pool_, 0, ac_group_sec.size(),
          prepare_storage, process_group, "DecodeGroup");
```

The `init_func` callback (e.g., `prepare_storage`, `resize_aux_outs`) is called once with `num_threads` and typically allocates per-thread scratch buffers. The `data_func` receives `(task_id, thread_id)` where `task_id` maps to a group index and `thread_id` identifies which thread's scratch space to use.

### DataParallel pattern

The init/data split serves a specific purpose: it separates allocation from processing. The init function runs on the main thread and can allocate per-thread storage (e.g., resizing a `vector<BitWriter>` to `num_threads` entries). The data function runs on worker threads and indexes into that storage via `thread_id`. This avoids both allocation in the hot loop and lock contention on shared data structures.

Error handling across threads uses an `atomic<uint32_t> has_error_` in `RunCallState`. If any task returns false, subsequent tasks see the error flag and return early.

### ThreadParallelRunner implementation

The default runner (`thread_parallel_runner_internal.cc`) uses a fork-join model with persistent worker threads:

- Worker threads are created at construction and block on `worker_start_cv_`.
- `Runner()` stores the function pointer and range, then signals all workers via `StartWorkers()`.
- Workers use a "guided" scheduling strategy (adapted from OpenMP): each thread atomically reserves `remaining / (4 * num_workers)` tasks at a time via `num_reserved_.fetch_add()`. This degrades gracefully -- large initial chunks reduce atomic contention, and the decreasing chunk size ensures good load balancing for stragglers.
- After all tasks are consumed, workers signal readiness via `workers_ready_cv_` and block again.
- False sharing is prevented with 64-byte padding around the `num_reserved_` atomic counter.
- Re-entrancy is explicitly detected and rejected (via `depth_` atomic counter).

## Dependencies

The SIMD and threading architecture depends on:

- **Highway** (`hwy/highway.h`, `hwy/foreach_target.h`): The SIMD abstraction layer. Provides the re-include mechanism, target dispatch, and all vector operations.
- **Standard C++ threading** (`std::thread`, `std::mutex`, `std::condition_variable`, `std::atomic`): Used by `ThreadParallelRunner`. No dependency on OpenMP or platform-specific threading APIs.
- **JxlMemoryManager**: Custom allocator interface used by the thread pool and pipeline stages for aligned allocations.
- **No SIMD intrinsics appear directly** in libjxl source. All SIMD is expressed through Highway's portable API. The only architecture-specific code is in Highway itself and in the `kRenderPipelineXOffset` constant (16 for ARM, 32 for x86) that ensures sufficient padding for the widest possible vector loads.

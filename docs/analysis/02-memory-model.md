# Memory Model

## Source Files

- `lib/jxl/image.h` (332 lines) — Defines `Plane<T>` (single-channel image) and `Image3<T>` (3-channel image) templates with aligned, padded row storage. Type aliases for common pixel types.
- `lib/jxl/image.cc` (108 lines) — Implements `PlaneBase` constructor, `Allocate()`, `Swap()`, and MSAN padding initialization.
- `lib/jxl/memory_manager_internal.h` (203 lines) — Defines alignment constants, `AlignedMemory` (aligned allocation wrapper), `AlignedArray<T>`, `MemoryManagerUniquePtr<T>`, and declares `BytesPerRow()`.
- `lib/jxl/memory_manager_internal.cc` (157 lines) — Implements `BytesPerRow()`, `AlignedMemory::Create()`, `AlignedMemory` constructor with rotation group logic, move/destructor, `MemoryManagerInit()`.
- `lib/include/jxl/memory_manager.h` (73 lines) — Public C API: `JxlMemoryManager` struct with `alloc`/`free` function pointers and opaque handle.
- `lib/jxl/base/data_parallel.h` (155 lines) — `ThreadPool` class wrapping `JxlParallelRunner`, plus `RunOnPool()` free function template.
- `lib/include/jxl/parallel_runner.h` (164 lines) — Public C API: `JxlParallelRunner` typedef, `JxlParallelRunInit`, `JxlParallelRunFunction` callback types.
- `lib/jxl/image_ops.h` (274 lines) — Operations on images: `CopyImageTo()`, `FillImage()`, `ZeroFillImage()`, `ScaleImage()`, `LinComb()`, `Mirror()`, `ImageMinMax()`, `DownsampleImage()`.
- `lib/jxl/padded_bytes.h` (205 lines) — `PaddedBytes`: growable byte buffer backed by `AlignedMemory`, used by `BitWriter`.
- `lib/jxl/base/rect.h` (186 lines) — `RectT<T>` template (aliased as `Rect = RectT<size_t>`): rectangular sub-region accessor into `Plane`/`Image3`.
- `lib/jxl/simd_util.h` (23 lines) — Declares `MaxVectorSize()` (runtime SIMD vector width query).
- `lib/jxl/simd_util.cc` (72 lines) — Implements `MaxVectorSize()` via Highway dynamic dispatch: `Lanes(HWY_FULL(float)) * sizeof(float)`.
- `lib/jxl/base/compiler_specific.h` (218 lines) — Compiler abstraction macros including `JXL_ASSUME_ALIGNED`, `JXL_RESTRICT`, `JXL_INLINE`.

## Key Types

### `detail::PlaneBase` (base class, non-template)

```cpp
namespace detail {
struct PlaneBase {
  // Fields (all non-const for move assignment):
  uint32_t xsize_;          // Valid pixel width (not including padding)
  uint32_t ysize_;          // Valid pixel height
  uint32_t orig_xsize_;     // Original allocated width (for ShrinkTo validation)
  uint32_t orig_ysize_;     // Original allocated height
  size_t bytes_per_row_;    // Row stride in bytes (includes padding)
  AlignedMemory bytes_;     // Owned aligned allocation
  size_t sizeof_t_;         // sizeof(T) for the component type

  // Construction
  PlaneBase();  // Zero-initializes all fields
  PlaneBase(uint32_t xsize, uint32_t ysize, size_t sizeof_t);  // Sets dims + computes bytes_per_row_

  // Copy forbidden, move defaulted
  PlaneBase(const PlaneBase&) = delete;
  PlaneBase& operator=(const PlaneBase&) = delete;
  PlaneBase(PlaneBase&&) noexcept = default;
  PlaneBase& operator=(PlaneBase&&) noexcept = default;

  // Key methods
  Status Allocate(JxlMemoryManager* memory_manager, size_t pre_padding);
  void Swap(PlaneBase& other);
  Status ShrinkTo(size_t xsize, size_t ysize);  // Requires xsize <= orig_xsize_, ysize <= orig_ysize_

  // Accessors
  size_t xsize() const;
  size_t ysize() const;
  size_t bytes_per_row() const;
  JxlMemoryManager* memory_manager() const;
  uint8_t* bytes();        // Returns JXL_ASSUME_ALIGNED(p, 64)
  const uint8_t* bytes() const;

protected:
  void* VoidRow(size_t y) const;  // bytes_ + y * bytes_per_row_, JXL_ASSUME_ALIGNED(row, 64)
};
}
```

Copy is explicitly deleted with comment: "to avoid inadvertent copies, which can be very expensive. Use CopyImageTo() instead."

### `Plane<T>` (single-channel image)

```cpp
template <typename ComponentType>
class Plane : public detail::PlaneBase {
public:
  using T = ComponentType;
  static constexpr size_t kNumPlanes = 1;

  Plane() = default;

  static StatusOr<Plane> Create(JxlMemoryManager* memory_manager,
                                size_t xsize, size_t ysize,
                                size_t pre_padding = 0);
  // static_assert: sizeof(T) must be 1, 2, 4, or 8
  // xsize/ysize are checked to fit in uint32_t

  T* Row(size_t y);              // static_cast<T*>(VoidRow(y))
  const T* Row(size_t y) const;
  const T* ConstRow(size_t y) const;

  ptrdiff_t PixelsPerRow() const;  // bytes_per_row_ / sizeof(T)
};
```

Allowed component sizes: 1, 2, 4, or 8 bytes (enforced by `static_assert`).

### Type Aliases

```cpp
using ImageSB = Plane<int8_t>;
using ImageB  = Plane<uint8_t>;
using ImageS  = Plane<int16_t>;   // signed integer or half-float
using ImageU  = Plane<uint16_t>;
using ImageI  = Plane<int32_t>;
using ImageF  = Plane<float>;     // THE primary image type throughout libjxl
using ImageD  = Plane<double>;
```

### `Image3<T>` (3-channel image)

```cpp
template <typename ComponentType>
class Image3 {
public:
  using T = ComponentType;
  using PlaneT = jxl::Plane<T>;
  static constexpr size_t kNumPlanes = 3;

  Image3();  // Default constructs 3 empty planes

  // Copy forbidden, move implemented (moves all 3 planes)
  Image3(const Image3&) = delete;
  Image3& operator=(const Image3&) = delete;
  Image3(Image3&&) noexcept;
  Image3& operator=(Image3&&) noexcept;

  static StatusOr<Image3> Create(JxlMemoryManager*, size_t xsize, size_t ysize);
  // Allocates 3 independent Plane<T> objects

  T* PlaneRow(size_t c, size_t y);
  const T* PlaneRow(size_t c, size_t y) const;
  const T* ConstPlaneRow(size_t c, size_t y) const;
  // PlaneRow optimization: computes row_offset = y * planes_[0].bytes_per_row()
  // once and reuses for any plane c, assuming all planes have same bytes_per_row

  const PlaneT& Plane(size_t idx) const;
  PlaneT& Plane(size_t idx);

  void Swap(Image3& other);
  Status ShrinkTo(size_t xsize, size_t ysize);

  // Delegates to planes_[0]:
  JxlMemoryManager* memory_manager() const;
  size_t xsize() const;
  size_t ysize() const;
  size_t bytes_per_row() const;
  ptrdiff_t PixelsPerRow() const;

private:
  Image3(PlaneT&&, PlaneT&&, PlaneT&&);
  PlaneT planes_[kNumPlanes];  // 3 independently allocated planes
};
```

### Type Aliases (3-channel)

```cpp
using Image3B = Image3<uint8_t>;
using Image3S = Image3<int16_t>;
using Image3U = Image3<uint16_t>;
using Image3I = Image3<int32_t>;
using Image3F = Image3<float>;    // THE primary 3-channel image type (linear float color)
using Image3D = Image3<double>;
```

### `AlignedMemory` (aligned allocation wrapper)

```cpp
class AlignedMemory {
public:
  AlignedMemory();  // nullptr state

  // Copy forbidden, custom move
  AlignedMemory(const AlignedMemory&) = delete;
  AlignedMemory& operator=(const AlignedMemory&) = delete;
  AlignedMemory(AlignedMemory&&) noexcept;
  AlignedMemory& operator=(AlignedMemory&&) noexcept;  // Frees existing if owned

  ~AlignedMemory();  // Frees via memory_manager_->free()

  static StatusOr<AlignedMemory> Create(JxlMemoryManager* memory_manager,
                                        size_t size, size_t pre_padding = 0);

  explicit operator bool() const noexcept;  // address_ != nullptr

  template <typename T>
  T* address() const;  // reinterpret_cast<T*>(address_)

  JxlMemoryManager* memory_manager() const;

private:
  AlignedMemory(JxlMemoryManager*, void* allocation, size_t pre_padding);

  void* allocation_;        // Raw pointer from alloc() — what gets passed to free()
  JxlMemoryManager* memory_manager_;
  void* address_;           // Aligned pointer within allocation_ (the usable address)
};
```

Three-pointer design: `allocation_` is what was returned by `alloc()` and must be passed to `free()`. `address_` is the aligned, rotated pointer the user actually works with. `memory_manager_` is set to `nullptr` on move-from to prevent double-free.

### `AlignedArray<T>`

```cpp
template <typename T>
class AlignedArray {
  size_t size_;
  AlignedMemory storage_;
  // Create() does placement-new for each element
  // Destructor calls ~T() for each element
  // operator[] with debug bounds check
};
```

### `MemoryManagerUniquePtr<T>`

```cpp
template <typename T>
using MemoryManagerUniquePtr = std::unique_ptr<T, MemoryManagerDeleteHelper>;
```

Custom deleter that calls `address->~T()` then `memory_manager_->free(opaque, address)`. Created via `MemoryManagerMakeUniquePrivate<T>()` which does placement-new after `alloc()`.

### `JxlMemoryManager` (public C API)

```cpp
typedef struct JxlMemoryManagerStruct {
  void* opaque;
  jpegxl_alloc_func alloc;   // void* (*)(void* opaque, size_t size)
  jpegxl_free_func free;     // void (*)(void* opaque, void* address)
} JxlMemoryManager;
```

Both `alloc` and `free` must be NULL (uses default malloc/free) or both non-NULL. Enforced by `MemoryManagerInit()`.

### `PaddedBytes` (growable aligned byte buffer)

```cpp
class PaddedBytes {
  JxlMemoryManager* memory_manager_;
  size_t size_;
  size_t capacity_;
  AlignedMemory data_;
  // reserve() allocates with +8 bytes for BitWriter overwrite
  // Growth factor: max(requested, 1.5x old), minimum 64 bytes
  // Copy forbidden, custom move
};
```

### `RectT<T>` / `Rect`

```cpp
template <typename T>
class RectT {
  T x0_, y0_;
  size_t xsize_, ysize_;
  // Row(image, y) → image->Row(y + y0_) + x0_
  // PlaneRow(image, c, y) → image->PlaneRow(c, y + y0_) + x0_
  // Note: Row() returns pointer offset by x0_ — NOT aligned to 64!
};
using Rect = RectT<size_t>;
```

Important: `Rect::Row()` adds `x0_` to the row pointer, so sub-region pointers are generally NOT 64-byte aligned. Only full-row pointers from `Plane::Row()` are guaranteed aligned.

### `ThreadPool`

```cpp
class ThreadPool {
  const JxlParallelRunner runner_;
  void* const runner_opaque_;

  template <class InitFunc, class DataFunc>
  Status Run(uint32_t begin, uint32_t end, const InitFunc& init_func,
             const DataFunc& data_func, const char* caller);

  static constexpr ThreadPoolNoInit NoInit{};
};
```

If `runner_` is NULL, runs sequentially on the calling thread with `num_threads=1`. The `RunCallState` inner class adapts C++ lambdas to C function pointer callbacks for `JxlParallelRunner`. Error propagation uses `std::atomic<uint32_t> has_error_`.

## Key Functions

### `BytesPerRow(size_t xsize, size_t sizeof_t)` (in memory_manager_internal.cc)

Computes row stride with three layers of padding:

```
1. valid_bytes = xsize * sizeof_t
2. If vec_size != 0: valid_bytes += vec_size - sizeof_t
   (allows unaligned SIMD load starting at last valid pixel)
3. align = max(vec_size, kAlignment)  // at least 128 bytes
   bytes_per_row = RoundUpTo(valid_bytes, align)
4. If bytes_per_row % kAlias == 0: bytes_per_row += align
   (Avoid2K: prevent store-to-load forwarding hazards on lower 11 address bits)
```

Returns 0 for xsize==0.

### `AlignedMemory::Create(JxlMemoryManager*, size_t size, size_t pre_padding)`

```
allocation_size = size + pre_padding + kAlias   // kAlias = 2048
```

Overflow check: if `size > allocation_size`, returns failure.

### `AlignedMemory` constructor (alignment + rotation)

```cpp
AlignedMemory(JxlMemoryManager*, void* allocation, size_t pre_padding) {
  static std::atomic<uint32_t> next_group{0};
  group = next_group.fetch_add(1, relaxed) & (kNumAlignmentGroups - 1);  // 0..15
  offset = kAlignment * group;  // 0, 128, 256, ..., 1920

  address = allocation + pre_padding;
  aligned_address = (address & ~(kAlias - 1)) + offset;  // Align to 2048 boundary + offset
  if (aligned_address < address) aligned_address += kAlias;
}
```

The rotation group is a global atomic counter. Each allocation gets a different offset within the 2 KiB alias window, cycling through 16 groups. This prevents cache set conflicts when multiple images of similar size are allocated consecutively.

### `PlaneBase::Allocate(JxlMemoryManager*, size_t pre_padding)`

```cpp
Status Allocate(...) {
  // Skips allocation for 0-dimensional images
  // Overflow check: ysize_ <= max_y_size (where max_y_size = SIZE_MAX / bytes_per_row_)
  bytes_ = AlignedMemory::Create(memory_manager, bytes_per_row_ * ysize_,
                                 pre_padding * sizeof_t_);
  InitializePadding(*this, sizeof_t_);  // MSAN only
}
```

### `InitializePadding()` (MSAN only, in image.cc)

Initializes the gap between `valid_size = xsize * sizeof_t` and `RoundUpTo(valid_size, vec_size)` in each row with `msan::kSanitizerSentinelByte`. This suppresses false MSAN warnings when SIMD loads read past the last valid pixel into padding.

### Row Accessors

- `PlaneBase::VoidRow(y)` — `bytes_.address<uint8_t>() + y * bytes_per_row_`, with `JXL_ASSUME_ALIGNED(row, 64)` and `JXL_DASSERT(y < ysize_)`
- `Plane<T>::Row(y)` — `static_cast<T*>(VoidRow(y))`
- `Plane<T>::ConstRow(y)` — same but returns `const T*`
- `Image3<T>::PlaneRow(c, y)` — `planes_[c].bytes() + y * planes_[0].bytes_per_row()`, cast to `T*` with `JXL_ASSUME_ALIGNED(row, 64)`. Uses `planes_[0].bytes_per_row()` for all planes (optimization: single multiply).
- `Rect::Row(image, y)` — `image->Row(y + y0_) + x0_` (NOT guaranteed aligned)

### Copy/Fill Operations (image_ops.h)

- `CopyImageTo(const Plane<T>&, Plane<T>*)` — row-by-row `memcpy` of `xsize * sizeof(T)` bytes
- `CopyImageTo(Rect, Plane<T>, Rect, Plane<T>*)` — rect-to-rect copy with bounds checks
- `CopyImageTo(Rect, Image3<T>, Rect, Image3<T>*)` — 3-plane rect copy
- `CopyImageTo(T, T*)` — whole-image convenience (delegates to rect version)
- `ZeroFillImage(Plane<T>*)` — row-by-row `memset(row, 0, xsize * sizeof(T))`
- `FillImage(T value, Plane<T>*)` — per-pixel fill
- `ScaleImage(T lambda, Plane<T>*)` — in-place multiply
- `LinComb(lambda1, image1, lambda2, image2)` — returns new plane

### `ShrinkTo(size_t xsize, size_t ysize)`

Allows reducing reported dimensions without reallocating. Does NOT recompute `bytes_per_row_` (would invalidate row contents). Used for pre-allocating with padding then reporting actual valid dimensions. Can also "un-shrink" up to `orig_xsize_`/`orig_ysize_`.

### `RunOnPool()`

```cpp
template <class InitFunc, class DataFunc>
Status RunOnPool(ThreadPool* pool, uint32_t begin, uint32_t end,
                 const InitFunc& init_func, const DataFunc& data_func,
                 const char* caller);
```

If `pool == nullptr`, creates a temporary single-threaded pool. Overload with `ThreadPoolNoInit` uses trivial init lambda.

## Constants

### Alignment Constants (memory_manager_internal.h, namespace `memory_manager_internal`)

```cpp
static constexpr size_t kAlignment = 2 * 64;  // 128 bytes
// Rationale: "To avoid RFOs, match L2 fill size (pairs of lines)"
// 2x cache line size. Must be power of 2.

static constexpr size_t kNumAlignmentGroups = 16;
// Number of distinct alignment offsets for rotation. Must be power of 2.

static constexpr size_t kAlias = kNumAlignmentGroups * kAlignment;  // 16 * 128 = 2048 bytes
// "Minimum multiple for which cache set conflicts and/or loads blocked
// by preceding stores can occur."
```

### Assumed Alignment for Row Pointers

All `bytes()`, `VoidRow()`, `Row()`, `PlaneRow()` return pointers annotated as `JXL_ASSUME_ALIGNED(ptr, 64)` (64-byte alignment hint to compiler). This is weaker than the actual allocation alignment (128 bytes minimum), providing a conservative guarantee.

### `hwy::kMaxVectorSize` (from Highway, compile-time)

```cpp
// third_party/highway/hwy/base.h:
static constexpr size_t kMaxVectorSize = 64;   // x86 (AVX-512)
static constexpr size_t kMaxVectorSize = 4096;  // SVE (scalable)
static constexpr size_t kMaxVectorSize = 16;   // other
```

Note: libjxl uses the runtime `MaxVectorSize()` for stride computation, not the compile-time `kMaxVectorSize`. The runtime value comes from Highway's dynamic dispatch (`HWY_FULL(float)` lanes times 4).

### `MaxVectorSize()` (runtime, simd_util.cc)

```cpp
size_t MaxVectorSize() {
  HWY_FULL(float) df;
  return Lanes(df) * sizeof(float);
  // Returns: 0 (scalar), 16 (SSE), 32 (AVX2), 64 (AVX-512), etc.
}
```

Returns 0 in scalar mode (no SIMD), in which case `BytesPerRow()` skips vector padding and just aligns to `kAlignment` (128 bytes).

## Memory Layout

### Planar Storage Model

Images are stored in **planar** format: each color channel is a separate contiguous allocation. An `Image3F` (3-channel float image) consists of 3 independent `Plane<float>` objects, each with its own `AlignedMemory` allocation.

```
Image3F:
  planes_[0] (R/X/Y channel): [row0 | pad | row1 | pad | row2 | pad | ...]
  planes_[1] (G/U/Cb channel): [row0 | pad | row1 | pad | row2 | pad | ...]
  planes_[2] (B/V/Cr channel): [row0 | pad | row1 | pad | row2 | pad | ...]
```

Each plane is a separate heap allocation. The 3 planes of an Image3 are NOT contiguous in memory.

### Row Layout Within a Plane

```
|<--- bytes_per_row_ --->|
[aligned pixel data][SIMD padding][alignment padding]
|<-- xsize*sizeof_t -->|<-- vec_size - sizeof_t -->|<-- roundup -->|

If bytes_per_row_ % 2048 == 0, an extra `align` bytes are added (Avoid2K).
```

For a concrete example, `ImageF` (float) with xsize=100, AVX2 (vec_size=32):

```
valid_bytes = 100 * 4 = 400
+ SIMD padding: 400 + 32 - 4 = 428
align = max(32, 128) = 128
bytes_per_row = RoundUpTo(428, 128) = 512
512 % 2048 != 0, so no Avoid2K adjustment
Final: bytes_per_row = 512 (128 pixels worth of stride for 100 valid pixels)
```

### Alignment Guarantee Chain

1. **Allocation**: `AlignedMemory::Create()` allocates `size + pre_padding + kAlias` (2048 extra bytes).
2. **Rotation**: The usable address is aligned to a `kAlias` (2048) boundary plus a rotating offset of `group * kAlignment` (group cycles 0..15, offset 0..1920 in steps of 128). This distributes allocations across 16 cache set groups.
3. **Row alignment**: Since `bytes_per_row_` is a multiple of `kAlignment` (128), and the base address is aligned to at least 128 bytes, every row start is at least 128-byte aligned.
4. **Compiler hint**: Row pointers are annotated as `JXL_ASSUME_ALIGNED(ptr, 64)` (conservative — actual alignment is higher).

### Cache Conflict Avoidance (Avoid2K)

The CPU's store buffer checks only the lower 11 bits of addresses for store-to-load forwarding hazards. If consecutive rows have the same lower 11 bits (i.e., `bytes_per_row` is a multiple of 2048), reading from row N while row N-1's writes are still in the store buffer triggers a false dependency stall. The `Avoid2K` logic adds one `align`-sized padding unit to break this.

The same problem occurs between planes of an `Image3`: if all three planes have the same alignment offset, the PlaneRow accessor (which computes `planes_[c].bytes() + y * bytes_per_row`) would produce addresses with identical lower bits. The allocation rotation (cycling through 16 groups) ensures different planes get different alignment offsets, preventing inter-plane cache set conflicts.

### Pre-padding

`Plane::Create()` accepts an optional `pre_padding` parameter (default 0). This is multiplied by `sizeof_t_` and passed to `AlignedMemory::Create()`. The `AlignedMemory` constructor places the aligned address at least `pre_padding` bytes after the raw allocation start. This allows negative-index access patterns (e.g., convolution kernels that read pixels before the start of a row from a previous row's padding).

Pre-padding is limited to `kAlias` (2048 bytes) by an `JXL_ENSURE` check.

## Algorithm Details

### Allocation Lifecycle

1. `Plane<T>::Create()` validates `sizeof(T)` and dimension limits, constructs a `PlaneBase` with computed `bytes_per_row_`, then calls `Allocate()`.
2. `Allocate()` creates an `AlignedMemory` with total size `bytes_per_row_ * ysize_` plus optional pre-padding.
3. `AlignedMemory::Create()` adds `kAlias` (2048) bytes to the requested size for alignment headroom, then calls `memory_manager->alloc()`.
4. The `AlignedMemory` constructor computes the aligned address using a rotating group counter to distribute allocations across cache sets.
5. Under MSAN, `InitializePadding()` writes sentinel bytes to the SIMD padding region of each row.
6. On destruction, `AlignedMemory::~AlignedMemory()` calls `memory_manager_->free(opaque, allocation_)` with the ORIGINAL allocation pointer (not the aligned address).

### Move Semantics

Both `Plane` and `Image3` support move but not copy. `AlignedMemory`'s move sets the source's `memory_manager_` to `nullptr` (not `allocation_`) to prevent double-free. Move-assignment of `AlignedMemory` frees the target's existing allocation first if it owns one.

### Thread Pool Execution Model

`ThreadPool::Run(begin, end, init_func, data_func, caller)`:

1. Calls `init_func(num_threads)` on the calling thread. If it returns false, returns error.
2. Calls `data_func(task, thread_id)` for each `task` in `[begin, end)`. Tasks may run in parallel across threads.
3. If any `data_func` returns false, sets `has_error_` (atomic) and subsequent calls return early.
4. Returns `JXL_FAILURE` if either init or any data call failed.

The `RunCallState` template adapts C++ lambdas to C-style function pointers (`JxlParallelRunInit`, `JxlParallelRunFunction`) that the external runner understands.

When `runner_` is NULL (no thread pool), tasks execute sequentially on thread 0 in order `[begin, end)`.

## Dependencies

- **Highway** (`<hwy/base.h>`, `<hwy/highway.h>`, `<hwy/foreach_target.h>`) — Provides `kMaxVectorSize`, `HWY_FULL`, `Lanes()`, `HWY_DYNAMIC_DISPATCH` for runtime SIMD width detection.
- **`<jxl/memory_manager.h>`** (public API) — `JxlMemoryManager` struct. User-provided or defaults to `malloc`/`free`.
- **`<jxl/parallel_runner.h>`** (public API) — `JxlParallelRunner` typedef. User-provided or NULL for single-threaded.
- **`lib/jxl/base/status.h`** — `Status`, `StatusOr<T>`, `JXL_RETURN_IF_ERROR`, `JXL_ENSURE`, `JXL_FAILURE` error handling macros.
- **`lib/jxl/base/compiler_specific.h`** — `JXL_ASSUME_ALIGNED`, `JXL_RESTRICT`, `JXL_INLINE`, `JXL_DASSERT`.
- **`lib/jxl/base/common.h`** — `RoundUpTo()`, `DivCeil()`.

## Open Questions

1. **`JXL_ASSUME_ALIGNED(ptr, 64)` vs actual alignment**: The actual alignment is at least 128 bytes (kAlignment), but the compiler hint says 64. Is this intentional conservatism, or could it be raised to 128 for better autovectorization?

2. **`Image3::PlaneRow` row_offset sharing**: The `PlaneRow(c, y)` method uses `planes_[0].bytes_per_row()` for the row offset calculation, assuming all three planes have the same stride. This is guaranteed by construction (all created with same xsize/sizeof_t), but there is no runtime assertion that the three planes actually have equal `bytes_per_row()`.

3. **Global atomic rotation counter**: `AlignedMemory`'s `next_group` is a `static std::atomic<uint32_t>`. This is a global counter shared across ALL allocations in the process. In a long-running process, this wraps around (mod 16). The rotation is best-effort, not guaranteed to avoid all conflicts.

4. **No cache-aligned alloc in public API**: The `JxlMemoryManager` has a TODO comment: `"Add cache-aligned alloc/free functions here."` Currently, alignment is achieved by over-allocating from the user's `alloc()` and adjusting the pointer internally. If a user provides a custom allocator that returns page-aligned memory, the kAlias overhead (2048 bytes per allocation) is wasted.

5. **`MaxVectorSize()` is runtime, called per-allocation**: `BytesPerRow()` calls `MaxVectorSize()` which does `HWY_DYNAMIC_DISPATCH`. This is called once per `Plane` construction. The Highway dispatch mechanism should be fast (function pointer), but this means row stride depends on which SIMD target was selected at runtime. Strides are not deterministic across different hardware.

6. **`pre_padding` use cases**: The `pre_padding` parameter on `Plane::Create()` defaults to 0. Need to trace callers to identify which image types actually use pre-padding and for what purpose (likely convolution border access).

7. **Ownership ambiguity** (noted in source comments at image.h:194-203): The codebase "abuses Image to either refer to an image that owns its storage or one that doesn't." A future redesign may split into a non-owning view class and a storage-holding `BackedImage`. Currently, `Plane` always owns its storage (no external-memory wrapping, despite the comment mentioning "deleter" support for wrapping).

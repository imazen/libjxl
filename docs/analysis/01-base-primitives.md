# Base Primitives

## Source Files

- `lib/jxl/base/compiler_specific.h` (218 lines) -- Compiler detection macros, portable attribute wrappers (`JXL_INLINE`, `JXL_NOINLINE`, `JXL_RESTRICT`, etc.), debug/sanitizer build configuration. The lowest-level header; everything else depends on it.
- `lib/jxl/base/sanitizer_definitions.h` (44 lines) -- Detects ASan, MSan, TSan via `__has_feature` or preprocessor defines. Sets `JXL_ADDRESS_SANITIZER`, `JXL_MEMORY_SANITIZER`, `JXL_THREAD_SANITIZER` to 0 or 1. Included by `compiler_specific.h`.
- `lib/jxl/base/common.h` (170 lines) -- Shared constants (`kBitsPerByte`, `kPi`, `kDefaultIntensityTarget`), arithmetic helpers (`DivCeil`, `RoundUpTo`, `Clamp1`), the `UninitializedAllocator` / `uninitialized_vector` type, `to_array` backport, `Color` typedef, `ToString` template, and the `JXL_JOIN` macro.
- `lib/jxl/base/status.h` (388 lines) -- Error handling: `StatusCode` enum, `Status` class (bool-like return type with error codes), `StatusOr<T>` (value-or-error), debug/logging macros (`JXL_DEBUG`, `JXL_FAILURE`, `JXL_RETURN_IF_ERROR`, `JXL_ASSIGN_OR_RETURN`, `JXL_DASSERT`, `JXL_ENSURE`, `JXL_UNREACHABLE`).
- `lib/jxl/base/span.h` (84 lines) -- `Span<T>` non-owning array view with `remove_prefix`, `AppendTo`, `Copy`. Also defines `Bytes = Span<const uint8_t>`.
- `lib/jxl/base/bits.h` (148 lines) -- Bit-manipulation intrinsics: CLZ, CTZ, floor/ceil log2, all portable across MSVC/GCC/Clang with `SizeTag` dispatch for 32-bit vs 64-bit overloads.

## Dependency Graph

```
sanitizer_definitions.h       (leaf -- no includes)
       |
       v
compiler_specific.h            (includes sanitizer_definitions.h)
       |
       v
common.h                       (includes compiler_specific.h)
       |
       v
status.h                       (includes common.h, compiler_specific.h)
      / \
     v   v
span.h  bits.h                 (both include status.h; bits.h also includes compiler_specific.h)
```

### Reverse dependency counts (files in `lib/jxl/` that include each header)

- `compiler_specific.h`: 176 files (effectively everything)
- `status.h`: 268 files (the most-included header in the project)
- `common.h`: 118 files
- `span.h`: 42 files
- `bits.h`: 28 files

## Key Types

### `StatusCode` (enum class : int32_t)

```cpp
enum class StatusCode : int32_t {
  kNotEnoughBytes = -1,  // Non-fatal error (negative)
  kOk = 0,              // The only success code
  kGenericError = 1,    // Fatal error (positive)
  kUnsupported = 2,     // Fatal error (positive)
};
```

Negative values = non-fatal (streaming can retry). Positive values = fatal (unrecoverable). Zero = success.

### `Status` (class, 4 bytes)

```cpp
class JXL_MUST_USE_RESULT Status {
  StatusCode code_;
public:
  constexpr Status(bool ok);            // true -> kOk, false -> kGenericError
  constexpr Status(StatusCode code);    // from enum
  constexpr operator bool() const;      // true iff code_ == kOk
  constexpr StatusCode code() const;
  constexpr bool IsFatalError() const;  // true iff code_ > 0
};
```

Implicit conversion from `bool` and to `bool`. `IsFatalError()` returns true for positive codes (kGenericError, kUnsupported), false for kOk and kNotEnoughBytes. The `JXL_MUST_USE_RESULT` attribute (`[[nodiscard]]`) ensures callers cannot silently drop a Status.

Static helper: `constexpr Status OkStatus()` returns `Status(StatusCode::kOk)`.

### `StatusOr<T>` (class template, move-only)

```cpp
template <typename T>
class JXL_MUST_USE_RESULT StatusOr {
  union Storage {
    char placeholder_;
    T data_;
  } storage_;
  StatusCode code_;
public:
  StatusOr(StatusCode code);    // error state (asserts code != kOk)
  StatusOr(Status status);      // delegates to StatusCode ctor
  StatusOr(T&& value);          // success state, placement-new into union
  StatusOr(StatusOr&& other);   // move ctor
  bool ok() const;
  Status status() const;
  T value_() &&;                // extracts value, asserts ok()
  ~StatusOr();                  // calls data_.~T() only if ok
};
```

Static assertions enforce that `T` is not convertible to/from `StatusCode`, and that `T` is move-constructible and move-assignable. Uses a union with a `char placeholder_` member so `T` is not default-constructed in the error path. Copy is deleted.

### `Span<T>` (class template, 16 bytes on 64-bit)

```cpp
template <typename T>
class Span {
  T* ptr_;
  size_t len_;
public:
  constexpr Span() noexcept;                          // nullptr, 0
  constexpr Span(T* array, size_t length) noexcept;
  explicit constexpr Span(T (&a)[N]) noexcept;        // from C array
  constexpr Span(U* array, size_t length) noexcept;   // reinterpret_cast, static_assert sizeof(U)==sizeof(T)
  explicit constexpr Span(const ArrayLike& other);     // from anything with .data()/.size()

  T* data() const;
  size_t size() const;
  bool empty() const;
  T* begin() const;
  T* end() const;
  T& operator[](size_t i) const;

  Status remove_prefix(size_t n);    // advances ptr, shrinks len, returns error if n > size
  void AppendTo(std::vector<NCT>& dst) const;
  std::vector<NCT> Copy() const;
};

using Bytes = Span<const uint8_t>;
```

The `reinterpret_cast` constructors are guarded by `static_assert(sizeof(U) == sizeof(T))` -- they allow byte-level reinterpretation but only between same-size types. `remove_prefix` uses `JXL_ENSURE` so it returns `Status` (fatal abort in debug, error propagation in release).

### `UninitializedAllocator<T>` (struct template)

```cpp
template <typename T>
struct UninitializedAllocator : std::allocator<T> {
  static_assert(std::is_trivially_copyable<T>::value, ...);
  template <typename U, typename... Args>
  void construct(U* place, Args&&... args) {}  // no-op: skips initialization
  template <typename U>
  void destroy(U* place) {}                    // no-op: skips destruction
};
```

Only valid for trivially copyable types. The `construct` no-op means `vector::resize()` skips zero-initialization, saving significant time for large buffers (image planes).

### `uninitialized_vector<T>` (type alias)

```cpp
template <typename T>
using uninitialized_vector = std::vector<T, UninitializedAllocator<T>>;
```

### `SizeTag<N>` (struct template, bits.h)

```cpp
template <size_t N>
struct SizeTag {};
```

Empty tag type used for compile-time dispatch between 32-bit (`SizeTag<4>`) and 64-bit (`SizeTag<8>`) overloads of bit intrinsics.

### `Color` (typedef, common.h)

```cpp
typedef std::array<float, 3> Color;
```

A 3-component float color value (no alpha). Used across color management, butteraugli, and image metadata.

## Key Functions

### Arithmetic (common.h)

- `RoundUpBitsToByteMultiple(size_t bits) -> size_t` -- `(bits + 7) & ~7`. Rounds bit count up to next byte boundary.
- `RoundUpToBlockDim(size_t dim) -> size_t` -- `(dim + 7) & ~7`. Rounds a dimension up to an 8-pixel block boundary. Identical formula to `RoundUpBitsToByteMultiple`.
- `DivCeil(T1 a, T2 b) -> T1` -- `(a + b - 1) / b`. Integer division rounding up. Constexpr, templated.
- `RoundUpTo(size_t what, size_t align) -> size_t` -- `DivCeil(what, align) * align`. Works for any alignment (not just power-of-two). Compiler optimizes to ADD+AND when `align` is power of two.
- `SafeAdd(uint64_t a, uint64_t b, uint64_t& sum) -> bool` -- Sets `sum = a + b`, returns true if no overflow. Overflow check: `sum >= a`.
- `Clamp1(T val, T low, T hi) -> T` -- Branchless-style ternary clamp. Marked `JXL_INLINE`.
- `Pi(T multiplier) -> T` -- Returns `multiplier * kPi` cast to type T.
- `ToString(T n) -> std::string` -- Converts integer or float to string via `snprintf`. Dispatches on `is_floating_point` / `is_unsigned`.

### Bit Manipulation (bits.h)

All marked `static JXL_INLINE JXL_MAYBE_UNUSED`. Undefined results for x == 0 unless noted.

- `Num0BitsAboveMS1Bit_Nonzero(SizeTag<4>, uint32_t x) -> size_t` -- Count leading zeros (CLZ) for 32-bit. Uses `__builtin_clz` (GCC/Clang) or `_BitScanReverse` (MSVC).
- `Num0BitsAboveMS1Bit_Nonzero(SizeTag<8>, uint64_t x) -> size_t` -- CLZ for 64-bit. Uses `__builtin_clzll` or `_BitScanReverse64`. On 32-bit MSVC (`!JXL_ARCH_X64`), splits into two 32-bit scans.
- `Num0BitsAboveMS1Bit_Nonzero(T x) -> size_t` -- Template wrapper, dispatches via `SizeTag<sizeof(T)>`. Static asserts `T` is unsigned.
- `Num0BitsBelowLS1Bit_Nonzero(SizeTag<4>, uint32_t x) -> size_t` -- Count trailing zeros (CTZ) for 32-bit. Uses `__builtin_ctz` or `_BitScanForward`.
- `Num0BitsBelowLS1Bit_Nonzero(SizeTag<8>, uint64_t x) -> size_t` -- CTZ for 64-bit. Same split strategy on 32-bit MSVC.
- `Num0BitsBelowLS1Bit_Nonzero(T x) -> size_t` -- Template wrapper with unsigned assertion.
- `Num0BitsAboveMS1Bit(T x) -> size_t` -- CLZ that handles zero: returns `sizeof(T) * 8` for x == 0.
- `Num0BitsBelowLS1Bit(T x) -> size_t` -- CTZ that handles zero: returns `sizeof(T) * 8` for x == 0.
- `FloorLog2Nonzero(T x) -> size_t` -- `(sizeof(T) * 8 - 1) ^ Num0BitsAboveMS1Bit_Nonzero(x)`. Undefined for x == 0. This is equivalent to the bit position of the highest set bit.
- `CeilLog2Nonzero(T x) -> size_t` -- `FloorLog2Nonzero(x)` if x is a power of two, otherwise `FloorLog2Nonzero(x) + 1`. Power-of-two check: `(x & (x - 1)) == 0`.
- `IsSigned<T>() -> bool` -- Constexpr. Returns true if `static_cast<T>(0) > static_cast<T>(-1)`.

### Error Construction (status.h)

- `Debug(const char* format, ...) -> bool` -- Always returns false. Prints to stderr (or android log). Marked `JXL_NOINLINE` to keep it out of hot paths. The return value lets it be used in comma expressions.
- `StatusMessage(Status status, const char* format, ...) -> Status` -- Prints debug message if `JXL_IS_DEBUG_BUILD && status.IsFatalError()` OR `JXL_DEBUG_ON_ALL_ERROR && !status`. Under `JXL_CRASH_ON_ERROR`, calls `Abort()` on fatal errors. Returns the input status unchanged.
- `Abort() -> [[noreturn]] bool` -- Debug-only. Prints stack trace (via sanitizer API if available), then calls `JXL_CRASH()` (`__builtin_trap()` or `__debugbreak()`).
- `OkStatus() -> Status` -- Returns `Status(StatusCode::kOk)`.

### Span Operations (span.h)

- `Span::remove_prefix(size_t n) -> Status` -- Advances `ptr_` by n, decrements `len_` by n. Uses `JXL_ENSURE(size() >= n)` which aborts in debug builds and returns `JXL_FAILURE` in release builds if `n > size()`.
- `Span::AppendTo(std::vector<NCT>& dst)` -- `dst.insert(dst.end(), begin(), end())`.
- `Span::Copy() -> std::vector<NCT>` -- Returns `std::vector<NCT>(begin(), end())`.

### Utility (common.h)

- `make_unique<T>(Args&&...)` -- Polyfill for C++11. Uses `std::make_unique` when C++14+ is available.
- `to_array(T (&&arr)[N]) -> std::array<remove_cv_t<T>, N>` -- Backport of `std::experimental::to_array`. Uses a custom `make_index_sequence` (not `std::index_sequence`) for C++11 compatibility.
- `make_uninitialized_vector<T>(size_t n) -> uninitialized_vector<T>` -- Factory that creates a vector of size `n` using `UninitializedAllocator`. Elements are NOT initialized.

## Constants

### common.h

- `kBitsPerByte = 8` -- Replaces `CHAR_BIT` for clarity.
- `kPi = 3.14159265358979323846264338327950288` -- Double precision pi. Used via `Pi(multiplier)` helper.
- `kInvLog2e = 0.6931471805599453` -- `1.0 / log2(e)` = `ln(2)`. Multiplier to convert log2 results to natural log.
- `kDefaultIntensityTarget = 255` -- Default display luminance in nits (cd/m^2). Shared between modules. Butteraugli was tuned for 250 nits; 255 is a practical round number for sRGB monitors.

### compiler_specific.h

- `JXL_CXX_17 = 201703` -- C++17 standard version constant.

### status.h

- `JXL_DEBUG_V_LEVEL = 0` (default) -- Compile-time verbose debug level. Messages with `level <= JXL_DEBUG_V_LEVEL` are printed; at 0, `JXL_DEBUG_V` compiles to nothing.

## Macros and Helpers

### Compiler Detection (compiler_specific.h)

- `JXL_COMPILER_MSVC` -- `_MSC_VER` if MSVC, else 0.
- `JXL_COMPILER_GCC` -- `__GNUC__ * 100 + __GNUC_MINOR__` if GCC (and NOT Clang), else 0. Clang undefines this because Clang pretends to be GCC.
- `JXL_COMPILER_CLANG` -- `__clang_major__ * 100 + __clang_minor__` if Clang, else 0.

### Portable Attributes (compiler_specific.h)

| Macro | MSVC | GCC/Clang |
|-------|------|-----------|
| `JXL_RESTRICT` | `__restrict` | `__restrict__` |
| `JXL_INLINE` | `__forceinline` | `inline __attribute__((always_inline))` |
| `JXL_NOINLINE` | `__declspec(noinline)` | `__attribute__((noinline))` |
| `JXL_NORETURN` | `__declspec(noreturn)` | `__attribute__((noreturn))` |
| `JXL_MAYBE_UNUSED` | (empty) | `__attribute__((unused))` |
| `JXL_LIKELY(expr)` | `expr` (no-op) | `__builtin_expect(!!(expr), 1)` |
| `JXL_UNLIKELY(expr)` | `expr` (no-op) | `__builtin_expect(!!(expr), 0)` |
| `JXL_ASSUME_ALIGNED(ptr, align)` | `(ptr)` (no-op) | `__builtin_assume_aligned((ptr), (align))` |
| `JXL_MUST_USE_RESULT` | (empty or `[[nodiscard]]`) | `[[nodiscard]]` or `__attribute__((warn_unused_result))` |
| `JXL_NO_SANITIZE(X)` | (empty) | `__attribute__((no_sanitize(X)))` |
| `JXL_FORMAT(idx_fmt, idx_arg)` | (empty) | `__attribute__((__format__(__printf__, idx_fmt, idx_arg)))` |
| `JXL_MAYBE_INLINE` | Under sanitizers: `JXL_MAYBE_UNUSED`; otherwise: `JXL_INLINE` | Same |

### Build Configuration (compiler_specific.h)

- `JXL_IS_DEBUG_BUILD` -- 1 unless `NDEBUG` is defined. Can be explicitly defined before include.
- `JXL_CRASH_ON_ERROR` -- 0 by default. If defined before include, set to 1. Requires `JXL_IS_DEBUG_BUILD`.
- `JXL_DEBUG_ON_ALL_ERROR` -- 0 by default. Prints debug messages on ALL error statuses (not just fatal). Requires `JXL_IS_DEBUG_BUILD`.
- `JXL_DEBUG_ON_ABORT` -- Defaults to `JXL_IS_DEBUG_BUILD`. Controls whether `JXL_DASSERT` and `JXL_ENSURE` print messages before aborting. Can be set to 0 to suppress.
- `JXL_CXX_LANG` -- `_MSVC_LANG` if MSVC (where `__cplusplus` lies), else `__cplusplus`.

### Sanitizer Detection (sanitizer_definitions.h)

- `JXL_ADDRESS_SANITIZER` -- 1 if ASan active, 0 otherwise. Checks `ADDRESS_SANITIZER` define or `__has_feature(address_sanitizer)`.
- `JXL_MEMORY_SANITIZER` -- 1 if MSan active, 0 otherwise.
- `JXL_THREAD_SANITIZER` -- 1 if TSan active, 0 otherwise.

### Stack Trace / Crash (compiler_specific.h)

- `JXL_PRINT_STACK_TRACE()` -- Calls `__sanitizer_print_stack_trace()` if any sanitizer is active; otherwise empty.
- `JXL_CRASH()` -- `__builtin_trap()` on GCC/Clang, `__debugbreak(), (void)abort()` on MSVC.

### Debug Logging (status.h)

- `JXL_DEBUG_TMP(format, ...)` -- Calls `Debug()` with `"%s:%d: " format "\n"` prepended (file + line).
- `JXL_DEBUG(enabled, format, ...)` -- Calls `JXL_DEBUG_TMP` only if `enabled` is true at compile time. Uses `do { ... } while(0)` wrapper.
- `JXL_DEBUG_V(level, format, ...)` -- Calls `JXL_DEBUG` with `level <= JXL_DEBUG_V_LEVEL` as the condition. When `JXL_DEBUG_V_LEVEL == 0` (default), this macro expands to nothing.
- `JXL_WARNING(format, ...)` -- `JXL_DEBUG(JXL_IS_DEBUG_BUILD, ...)`. Prints only in debug builds.

### Error Macros (status.h)

- `JXL_STATUS(status, format, ...)` -- Calls `StatusMessage()` which conditionally prints and returns the status.
- `JXL_NOTIFY_ERROR(format, ...)` -- Calls `JXL_STATUS` with `kGenericError`, discards the result (`(void)`). Only useful for side effects (debug output).
- `JXL_FAILURE(format, ...)` -- Prints debug message via `JXL_STATUS`, then returns `Status(kGenericError)`. Uses the comma operator: `((void)JXL_STATUS(...), Status(kGenericError))`.
- `JXL_UNSUPPORTED(format, ...)` -- Same pattern, returns `Status(kUnsupported)`.
- `JXL_NOT_ENOUGH_BYTES(format, ...)` -- Same pattern, returns `Status(kNotEnoughBytes)`.
- `JXL_RETURN_IF_ERROR(status)` -- Evaluates `status` exactly once into a local variable `jxl_return_if_error_status`. If not ok, prints debug info (file, line, code, expression string) and returns the status. This is the primary error propagation mechanism.
- `JXL_QUIET_RETURN_IF_ERROR(status)` -- Same as `JXL_RETURN_IF_ERROR` but without calling `StatusMessage`. Used in `fields.h` bundles where numerous call sites would generate excessive messages during partial header decode.

### Assertions (status.h)

- `JXL_DASSERT(condition)` -- Debug-only assert. In debug: prints condition string and calls `Abort()`. In release: compiles to nothing.
- `JXL_ENSURE(condition)` -- In debug: prints and aborts (like `JXL_DASSERT`). In release: returns `JXL_FAILURE("JXL_ENSURE: %s", #condition)`. This means functions using `JXL_ENSURE` must return `Status`.
- `JXL_DEBUG_ABORT(format, ...)` -- Debug-only. Prints message, then calls `Abort()`. In release: compiles to nothing.
- `JXL_UNREACHABLE(format, ...)` -- In debug: prints, aborts, AND evaluates `JXL_FAILURE` (for type consistency). In release: returns `JXL_FAILURE("internal: " format)`.

### StatusOr Helpers (status.h)

- `JXL_ASSIGN_OR_RETURN(lhs, statusor)` -- Evaluates `statusor` into a uniquely-named temp variable (using `JXL_JOIN` + `__LINE__`), calls `JXL_RETURN_IF_ERROR(name.status())` to propagate errors, then assigns `std::move(name).value_()` to `lhs`.
- `JXL_ASSIGN_OR_QUIT(lhs, statusor, message)` -- Same pattern but calls `QUIT(message)` instead of returning. Used in contexts that cannot propagate `Status` (e.g., `main()`).
- `JXL_JOIN(x, y)` / `JXL_DO_JOIN(x, y)` -- Standard two-level token paste macro for generating unique variable names.

## Error Handling Pattern

The error handling follows a consistent pattern throughout libjxl:

1. **Functions return `Status`** instead of `bool`. `Status` implicitly converts from/to `bool`, so `return true` and `return false` work naturally, but `Status` carries richer error codes and is `[[nodiscard]]`.

2. **Error propagation** uses `JXL_RETURN_IF_ERROR(expr)`:
   ```cpp
   Status DoSomething() {
     JXL_RETURN_IF_ERROR(SubStep1());
     JXL_RETURN_IF_ERROR(SubStep2());
     return true;
   }
   ```
   The macro evaluates the expression exactly once, checks for error, and early-returns the same `Status` (preserving the error code). Debug builds print file:line and the expression text.

3. **Value-or-error** uses `StatusOr<T>` with `JXL_ASSIGN_OR_RETURN`:
   ```cpp
   StatusOr<Widget> MakeWidget();

   Status UseWidget() {
     Widget w;
     JXL_ASSIGN_OR_RETURN(w, MakeWidget());
     // w is now valid
     return true;
   }
   ```

4. **Three error severities**:
   - `kNotEnoughBytes` (-1): non-fatal, streaming decoder can retry with more data. `IsFatalError()` returns false.
   - `kGenericError` (1): fatal, something is wrong with the input or internal state.
   - `kUnsupported` (2): fatal, feature not implemented.

5. **Debug vs release behavior**:
   - Debug: `JXL_DASSERT` aborts, `JXL_ENSURE` aborts, `JXL_FAILURE` prints to stderr.
   - Release: `JXL_DASSERT` is no-op, `JXL_ENSURE` returns error status, `JXL_FAILURE` is silent (unless `JXL_DEBUG_ON_ALL_ERROR`).
   - `JXL_CRASH_ON_ERROR` (debug only): `Abort()` on any fatal error, turning status returns into hard crashes for debugging.

## Algorithm Details

### Bit Intrinsic Dispatch (bits.h)

The CLZ/CTZ functions use tag dispatch (`SizeTag<4>` vs `SizeTag<8>`) rather than `if constexpr` or SFINAE, which keeps the code C++11 compatible. The 32-bit MSVC path for 64-bit operations splits the value into upper/lower 32-bit halves and uses `_BitScanReverse`/`_BitScanForward` on each, since `_BitScanReverse64` and `_BitScanForward64` are not available on 32-bit x86 MSVC.

`FloorLog2Nonzero` uses the identity: for nonzero x, `floor(log2(x))` equals the index of the highest set bit, which is `(bit_width - 1) - clz(x)`. The XOR with `(sizeof(T) * 8 - 1)` is equivalent to subtraction from that constant when the result is in range `[0, bit_width - 1]`.

### UninitializedAllocator (common.h)

The allocator exploits the STL allocator interface: by making `construct()` a no-op, `vector::resize(n)` allocates memory but does not zero-fill it. This is safe only because the `static_assert` restricts `T` to trivially copyable types (where skipping construction and destruction is well-defined). The `destroy()` no-op matches -- trivially copyable types have trivial destructors.

### StatusOr Storage (status.h)

`StatusOr<T>` uses a union with a `char placeholder_` to avoid default-constructing `T` when the StatusOr holds an error. The `T` is placement-new'd into `storage_.data_` only in the success path. The destructor conditionally calls `storage_.data_.~T()` only if `code_ == kOk`. This is a manual discriminated union, equivalent to what `std::variant` or `std::optional` do internally, but without the C++17 dependency.

The move constructor and move assignment operator handle all four cases (both ok, only source ok, only dest ok, neither ok) to properly manage the union lifetime.

## Dependencies

### Depends on (external)

- Standard C headers: `<cstdarg>`, `<cstdint>`, `<cstdio>`, `<cstdlib>`, `<stddef.h>`, `<stdint.h>`, `<sys/types.h>`
- Standard C++ headers: `<array>`, `<memory>`, `<string>`, `<type_traits>`, `<utility>`, `<vector>`
- Sanitizer headers (conditional): `<sanitizer/common_interface_defs.h>`, `<android/log.h>`
- MSVC intrinsics (conditional): `<intrin.h>`

### Depended on by

These five files form the foundation of the entire libjxl codebase. `status.h` alone is included by 268 files. Every module in the encoder, decoder, render pipeline, modular coding, JPEG transcoding, color management, and test infrastructure depends on these base primitives. Notable direct dependents include:

- `lib/jxl/base/rect.h`, `lib/jxl/base/float.h`, `lib/jxl/base/data_parallel.h`, `lib/jxl/base/sanitizers.h`, `lib/jxl/base/random.h` -- other base layer files
- `lib/jxl/image.h` -- the `ImageF` / `Image3F` types
- `lib/jxl/fields.h` -- bundle serialization (uses `JXL_QUIET_RETURN_IF_ERROR`)
- `lib/jxl/dec_bit_reader.h` -- bit-level stream reading (uses `bits.h`, `span.h`)
- `lib/jxl/dec_ans.h` -- ANS entropy decoder (uses `bits.h`)
- `lib/jxl/enc_bit_writer.h` -- bit-level stream writing (uses `span.h`)

## Open Questions

- `StatusCode` only has 4 values. Is there a reason it uses `int32_t` rather than a smaller type? Possibly for ABI stability or alignment.
- `JXL_UNREACHABLE` in debug mode evaluates `JXL_FAILURE` after `Abort()` -- `Abort()` is `[[noreturn]]`, so the `JXL_FAILURE` is dead code. It appears to exist solely to satisfy return type checking in expressions.
- The `Span` reinterpret_cast constructors are potentially UB under strict aliasing rules, though they are guarded by the `sizeof` check. In practice they are used for byte-level reinterpretation which is common in codec code.
- `IsSigned<T>()` in bits.h uses a comparison trick (`static_cast<T>(0) > static_cast<T>(-1)`) rather than `std::is_signed<T>::value`. The comment says "TODO(eustas): remove dupes", suggesting this may be redundant with standard type traits.
- `UninitializedAllocator::construct` takes `Args&&... args` and ignores them. This means even explicit constructor arguments are silently discarded. This is safe for the intended use case (resize/reserve with trivially copyable types) but could be surprising in other contexts.
- `JXL_ASSIGN_OR_QUIT` references a `QUIT(message)` macro that is not defined in any of these base files -- it must be defined by the tool/binary that uses it.

# OnYX Coding Standards

**Version:** 1.0  
**Effective:** February 2026  
**Applies to:** All OnYX production code

---

## Table of Contents

1. [Philosophy](#1-philosophy)
2. [SOLID Principles](#2-solid-principles)
3. [Naming Conventions](#3-naming-conventions)
4. [Code Formatting](#4-code-formatting)
5. [File Organisation](#5-file-organisation)
6. [Documentation](#6-documentation)
7. [Error Handling](#7-error-handling)
8. [Memory Management](#8-memory-management)
9. [CUDA Conventions](#9-cuda-conventions)
10. [Testing Standards](#10-testing-standards)
11. [Version Control](#11-version-control)
12. [Code Review Checklist](#12-code-review-checklist)

---

## 1. Philosophy

### 1.1 Core Principles

OnYX code must be:

- **Correct** — Produces right results for all valid inputs
- **Robust** — Handles invalid inputs gracefully without crashing
- **Readable** — Intent is clear to future maintainers
- **Maintainable** — Changes can be made safely and locally
- **Testable** — Can be verified in isolation
- **Performant** — Meets real-time video processing requirements

### 1.2 Priority Order

When principles conflict, resolve in this order:

1. **Safety** — No crashes, no undefined behaviour, no data corruption
2. **Correctness** — Right output for valid input
3. **Clarity** — Code that explains itself
4. **Performance** — Speed, but only after the above are satisfied

### 1.3 The Boy Scout Rule

Leave code cleaner than you found it. Every commit should improve quality.

---

## 2. SOLID Principles

All OnYX code adheres to SOLID principles. These are not optional guidelines — they are requirements.

### 2.1 Single Responsibility Principle (SRP)

> A class should have only one reason to change.

**Requirement:** Each class handles exactly one concern.

**Good:**
```cpp
class ButterworthFilter {
    // Only responsibility: Apply Butterworth filtering
public:
    BiquadCoeffs designLowpass(double fc, double fs);
    void apply(std::vector<double>& signal, const BiquadCoeffs& coeffs);
    void applyZeroPhase(std::vector<double>& signal, const BiquadCoeffs& coeffs);
};

class MotionPathSmoother {
    // Only responsibility: Smooth motion paths using filters
public:
    void smooth(MotionPath& path, const SmoothingParams& params);
private:
    ButterworthFilter filter_;  // Uses filter, doesn't implement it
};
```

**Bad:**
```cpp
class MotionPathSmoother {
    // VIOLATION: Multiple responsibilities mixed together
public:
    void smooth(MotionPath& path);
    BiquadCoeffs designFilter(double fc);  // Filter design doesn't belong here
    void writeCSV(const std::string& path); // I/O doesn't belong here
    void renderFrame(Frame& f);             // Rendering doesn't belong here
};
```

**Litmus test:** Can you describe the class without using "and"?

### 2.2 Open/Closed Principle (OCP)

> Software entities should be open for extension but closed for modification.

**Requirement:** Add new behaviour by adding code, not changing existing code.

**Good:**
```cpp
// Base smoother interface — closed for modification
class ISmoother {
public:
    virtual ~ISmoother() = default;
    virtual void smooth(MotionPath& path) = 0;
    virtual std::string name() const = 0;
};

// Extensions — open for extension via new classes
class ButterworthSmoother : public ISmoother {
public:
    void smooth(MotionPath& path) override;
    std::string name() const override { return "Butterworth"; }
};

class HessianSmoother : public ISmoother {
public:
    void smooth(MotionPath& path) override;
    std::string name() const override { return "Hessian"; }
};

class AdaptiveSmoother : public ISmoother {
public:
    void smooth(MotionPath& path) override;
    std::string name() const override { return "Adaptive"; }
};

// Adding a new smoother requires NO changes to existing code
class KalmanSmoother : public ISmoother { /* ... */ };
```

**Bad:**
```cpp
void smooth(MotionPath& path, SmootherType type) {
    // VIOLATION: Adding new smoother requires modifying this function
    switch (type) {
        case SmootherType::Butterworth: /* ... */ break;
        case SmootherType::Hessian:     /* ... */ break;
        case SmootherType::Adaptive:    /* ... */ break;
        // Must modify to add new type
    }
}
```

### 2.3 Liskov Substitution Principle (LSP)

> Subtypes must be substitutable for their base types.

**Requirement:** Derived classes must honour the contract of their base class.

**Good:**
```cpp
class ISmoother {
public:
    // Contract: smooth() modifies path in-place, does not throw for valid input
    virtual void smooth(MotionPath& path) = 0;
};

class ButterworthSmoother : public ISmoother {
public:
    // Honors contract: modifies path, no exceptions for valid input
    void smooth(MotionPath& path) override {
        if (path.empty()) return;  // Valid no-op for edge case
        // ... apply filtering ...
    }
};
```

**Bad:**
```cpp
class BrokenSmoother : public ISmoother {
public:
    void smooth(MotionPath& path) override {
        // VIOLATION: Throws where base class doesn't specify it can throw
        throw std::runtime_error("Not implemented");
    }
};

class AnotherBrokenSmoother : public ISmoother {
public:
    void smooth(MotionPath& path) override {
        // VIOLATION: Returns new path instead of modifying in-place
        // Caller expects path to be modified
        MotionPath newPath = computeSmoothed(path);
        // path unchanged — contract violated
    }
};
```

**Litmus test:** Can you replace any instance of the base class with the derived class without breaking the program?

### 2.4 Interface Segregation Principle (ISP)

> Clients should not be forced to depend on interfaces they do not use.

**Requirement:** Prefer small, focused interfaces over large, general ones.

**Good:**
```cpp
// Segregated interfaces — clients depend only on what they need
class IAnalyser {
public:
    virtual ~IAnalyser() = default;
    virtual MotionPath analyse(const FrameSequence& frames) = 0;
};

class IRenderer {
public:
    virtual ~IRenderer() = default;
    virtual void render(const Frame& src, Frame& dst, const Transform& t) = 0;
};

class ISmoother {
public:
    virtual ~ISmoother() = default;
    virtual void smooth(MotionPath& path) = 0;
};

// PhaseCorrelator only implements what it does
class PhaseCorrelator : public IAnalyser {
public:
    MotionPath analyse(const FrameSequence& frames) override;
};
```

**Bad:**
```cpp
// VIOLATION: Monolithic interface forces implementers to provide everything
class IStabiliser {
public:
    virtual MotionPath analyse(const FrameSequence& frames) = 0;
    virtual void smooth(MotionPath& path) = 0;
    virtual void render(const Frame& src, Frame& dst) = 0;
    virtual void writeCSV(const std::string& path) = 0;
    virtual void loadLicense(const std::string& key) = 0;
};

// Implementer forced to provide methods it doesn't need
class SimpleRenderer : public IStabiliser {
    MotionPath analyse(...) override { throw; }  // Doesn't analyse
    void smooth(...) override { throw; }          // Doesn't smooth
    void render(...) override { /* actual impl */ }
    void writeCSV(...) override { throw; }        // Doesn't write CSV
    void loadLicense(...) override { throw; }     // Doesn't handle licenses
};
```

### 2.5 Dependency Inversion Principle (DIP)

> High-level modules should not depend on low-level modules. Both should depend on abstractions.

**Requirement:** Depend on interfaces, not concrete implementations.

**Good:**
```cpp
// High-level policy depends on abstraction
class StabilisationPipeline {
public:
    StabilisationPipeline(
        std::unique_ptr<IAnalyser> analyser,
        std::unique_ptr<ISmoother> smoother,
        std::unique_ptr<IRenderer> renderer
    ) : analyser_(std::move(analyser)),
        smoother_(std::move(smoother)),
        renderer_(std::move(renderer)) {}

    void process(FrameSequence& frames) {
        MotionPath path = analyser_->analyse(frames);
        smoother_->smooth(path);
        for (size_t i = 0; i < frames.size(); ++i) {
            renderer_->render(frames[i], output_[i], path.transformAt(i));
        }
    }

private:
    std::unique_ptr<IAnalyser> analyser_;
    std::unique_ptr<ISmoother> smoother_;
    std::unique_ptr<IRenderer> renderer_;
};

// Construction with concrete types happens at composition root
auto pipeline = std::make_unique<StabilisationPipeline>(
    std::make_unique<PhaseCorrelator>(),
    std::make_unique<ButterworthSmoother>(),
    std::make_unique<CudaRenderer>()
);
```

**Bad:**
```cpp
// VIOLATION: High-level class creates its own dependencies
class StabilisationPipeline {
public:
    StabilisationPipeline() {
        // Hardcoded dependency — cannot test, cannot extend
        analyser_ = std::make_unique<PhaseCorrelator>();
        smoother_ = std::make_unique<ButterworthSmoother>();
        renderer_ = std::make_unique<CudaRenderer>();
    }
};
```

---

## 3. Naming Conventions

### 3.1 General Rules

- Names describe **what**, not **how**
- Prefer clarity over brevity
- Avoid abbreviations except universally understood ones (e.g., `fps`, `id`, `max`, `min`)
- Avoid Hungarian notation (no `strName`, `iCount`, `pPointer`)

### 3.2 Specific Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Classes | PascalCase | `MotionPath`, `ButterworthFilter` |
| Interfaces | PascalCase with I prefix | `IAnalyser`, `ISmoother` |
| Functions/Methods | camelCase | `analyseFrames()`, `computeHolonomy()` |
| Variables | camelCase | `frameCount`, `maxDisplacement` |
| Member variables | camelCase with trailing underscore | `path_`, `coefficients_` |
| Constants | SCREAMING_SNAKE_CASE | `MAX_FRAMES`, `DEFAULT_CUTOFF` |
| Namespaces | lowercase | `onyx`, `onyx::cuda` |
| Files | PascalCase matching class | `ButterworthFilter.cpp`, `ButterworthFilter.h` |
| CUDA kernels | PascalCase with Kernel suffix | `RenderKernel`, `LumaKernel` |

### 3.3 Meaningful Names

```cpp
// Good — names reveal intent
double computeHolonomy(const std::vector<Vec2>& velocities, int windowSize);
bool isLicenseValid(const License& license);
void applyZeroPhaseFilter(std::vector<double>& signal);

// Bad — names obscure intent
double calc(const std::vector<Vec2>& v, int n);  // calc what?
bool check(const License& l);                      // check what?
void process(std::vector<double>& s);              // process how?
```

### 3.4 Booleans

Boolean variables and functions should read as questions:

```cpp
// Good
bool isValid;
bool hasLicense;
bool shouldApplyZoom;
bool isEmpty();
bool canProcess();

// Bad
bool valid;      // valid what?
bool license;    // license what?
bool zoom;       // zoom what?
bool check();    // check what?
```

---

## 4. Code Formatting

### 4.1 Indentation and Braces

- **Indentation:** 4 spaces (no tabs)
- **Braces:** Allman style (opening brace on new line)
- **Line length:** 100 characters maximum

```cpp
namespace onyx
{

class ButterworthFilter
{
public:
    ButterworthFilter(double sampleRate);

    BiquadCoeffs designLowpass(double cutoffHz);

    void apply(std::vector<double>& signal, const BiquadCoeffs& coeffs);

private:
    double sampleRate_;
};

void ButterworthFilter::apply(std::vector<double>& signal, const BiquadCoeffs& coeffs)
{
    if (signal.empty())
    {
        return;
    }

    for (size_t i = 0; i < signal.size(); ++i)
    {
        // Process sample
    }
}

} // namespace onyx
```

### 4.2 Spacing

```cpp
// Around operators
int result = a + b * c;
bool valid = (x > 0) && (y < max);

// After keywords
if (condition)
while (running)
for (int i = 0; i < n; ++i)

// No space before parentheses in function calls
computeHolonomy(velocities, windowSize);

// Space after commas
void process(int a, int b, int c);
```

### 4.3 Vertical Spacing

- One blank line between functions
- Two blank lines between sections (public/private, class definitions)
- No multiple consecutive blank lines
- No trailing whitespace

### 4.4 Include Order

```cpp
// 1. Corresponding header (for .cpp files)
#include "ButterworthFilter.h"

// 2. C system headers
#include <cmath>
#include <cstdint>

// 3. C++ standard library headers
#include <algorithm>
#include <memory>
#include <vector>

// 4. Third-party library headers
#include <cuda_runtime.h>
#include <ofxImageEffect.h>

// 5. Project headers
#include "MotionPath.h"
#include "Transform.h"
```

---

## 5. File Organisation

### 5.1 Directory Structure

```
onyx/
├── src/
│   ├── core/                 # Core algorithms (platform-independent)
│   │   ├── ButterworthFilter.h
│   │   ├── ButterworthFilter.cpp
│   │   ├── HolonomyCalculator.h
│   │   ├── HolonomyCalculator.cpp
│   │   ├── MotionPath.h
│   │   ├── MotionPath.cpp
│   │   ├── SpikeFilter.h
│   │   ├── SpikeFilter.cpp
│   │   └── ...
│   ├── cuda/                 # CUDA kernels and wrappers
│   │   ├── CudaContext.h
│   │   ├── CudaContext.cpp
│   │   ├── Kernels.h
│   │   ├── Kernels.cu
│   │   ├── PhaseCorrelator.h
│   │   ├── PhaseCorrelator.cpp
│   │   └── ...
│   ├── ofx/                  # OFX plugin interface
│   │   ├── OnyxPlugin.h
│   │   ├── OnyxPlugin.cpp
│   │   ├── ParameterManager.h
│   │   ├── ParameterManager.cpp
│   │   └── ...
│   ├── licensing/            # License validation
│   │   ├── License.h
│   │   ├── License.cpp
│   │   ├── MachineFingerprint.h
│   │   ├── MachineFingerprint.cpp
│   │   └── ...
│   └── util/                 # Shared utilities
│       ├── Logger.h
│       ├── Logger.cpp
│       ├── Result.h
│       └── ...
├── tests/
│   ├── core/
│   ├── cuda/
│   └── ...
├── docs/
├── cmake/
└── CMakeLists.txt
```

### 5.2 Header Files

```cpp
// ButterworthFilter.h
#ifndef ONYX_CORE_BUTTERWORTH_FILTER_H
#define ONYX_CORE_BUTTERWORTH_FILTER_H

#include <vector>

namespace onyx
{
namespace core
{

/**
 * @brief Second-order Butterworth filter coefficients.
 */
struct BiquadCoeffs
{
    double b0, b1, b2;  // Numerator (feedforward)
    double a1, a2;      // Denominator (feedback), a0 = 1.0
};

/**
 * @brief Butterworth lowpass filter design and application.
 *
 * Implements second-order IIR Butterworth filters using the bilinear
 * transform. Supports both causal and zero-phase (forward-backward)
 * filtering.
 */
class ButterworthFilter
{
public:
    /**
     * @brief Construct filter for given sample rate.
     * @param sampleRate Sample rate in Hz. Must be positive.
     * @throws std::invalid_argument if sampleRate <= 0.
     */
    explicit ButterworthFilter(double sampleRate);

    /**
     * @brief Design lowpass filter coefficients.
     * @param cutoffHz Cutoff frequency in Hz. Must be in (0, sampleRate/2).
     * @return Biquad coefficients for the filter.
     * @throws std::invalid_argument if cutoffHz out of valid range.
     */
    BiquadCoeffs designLowpass(double cutoffHz) const;

    /**
     * @brief Apply zero-phase filtering (forward-backward).
     * @param signal Signal to filter (modified in-place).
     * @param coeffs Filter coefficients from designLowpass().
     */
    void applyZeroPhase(std::vector<double>& signal, const BiquadCoeffs& coeffs) const;

private:
    double sampleRate_;

    void applyForward(std::vector<double>& signal, const BiquadCoeffs& coeffs) const;
    void applyBackward(std::vector<double>& signal, const BiquadCoeffs& coeffs) const;
};

} // namespace core
} // namespace onyx

#endif // ONYX_CORE_BUTTERWORTH_FILTER_H
```

### 5.3 Source Files

```cpp
// ButterworthFilter.cpp
#include "ButterworthFilter.h"

#include <cmath>
#include <stdexcept>
#include <algorithm>

namespace onyx
{
namespace core
{

namespace
{
    constexpr double PI = 3.14159265358979323846;
} // anonymous namespace


ButterworthFilter::ButterworthFilter(double sampleRate)
    : sampleRate_(sampleRate)
{
    if (sampleRate <= 0.0)
    {
        throw std::invalid_argument("Sample rate must be positive");
    }
}

BiquadCoeffs ButterworthFilter::designLowpass(double cutoffHz) const
{
    if (cutoffHz <= 0.0 || cutoffHz >= sampleRate_ / 2.0)
    {
        throw std::invalid_argument("Cutoff must be in (0, sampleRate/2)");
    }

    // Bilinear transform pre-warp
    const double wc = 2.0 * PI * cutoffHz;
    const double T = 1.0 / sampleRate_;
    const double wa = (2.0 / T) * std::tan(wc * T / 2.0);

    // ... rest of implementation
}

// ... other methods

} // namespace core
} // namespace onyx
```

---

## 6. Documentation

### 6.1 File Headers

Every source file begins with:

```cpp
/**
 * @file ButterworthFilter.cpp
 * @brief Butterworth lowpass filter implementation.
 *
 * @copyright 2026 Baker & Boyle Research
 * @license Proprietary
 */
```

### 6.2 Class Documentation

```cpp
/**
 * @brief Computes holonomy measure for motion paths.
 *
 * Holonomy measures the degree to which a motion path curves versus
 * translates linearly. High holonomy indicates intentional camera
 * movement (pans, tilts); low holonomy indicates unwanted shake.
 *
 * The implementation uses a sliding window approach with configurable
 * window size. Computational complexity is O(n) where n is path length.
 *
 * @see MotionPath
 * @see AdaptiveSmoother
 */
class HolonomyCalculator
{
    // ...
};
```

### 6.3 Function Documentation

```cpp
/**
 * @brief Compute holonomy for a velocity sequence.
 *
 * Holonomy is computed as the ratio of path length to displacement:
 *   H = pathLength / displacement
 *
 * For a straight-line path, H = 1.0. For curved paths, H > 1.0.
 *
 * @param velocities Per-frame velocity vectors.
 * @param windowSize Sliding window size in frames. Must be >= 3.
 * @return Holonomy values for each frame. Vector length equals input length.
 *         Edge frames use reduced window sizes.
 *
 * @throws std::invalid_argument if windowSize < 3.
 * @throws std::invalid_argument if velocities is empty.
 *
 * @note Thread-safe: This function has no side effects.
 *
 * @code
 * HolonomyCalculator calc;
 * auto holonomy = calc.compute(velocities, 51);
 * double meanH = std::accumulate(holonomy.begin(), holonomy.end(), 0.0) / holonomy.size();
 * @endcode
 */
std::vector<double> compute(const std::vector<Vec2>& velocities, int windowSize) const;
```

### 6.4 Inline Comments

Use inline comments sparingly to explain **why**, not **what**:

```cpp
// Good — explains non-obvious reasoning
// Use forward-backward filtering to achieve zero phase distortion.
// Single-pass filtering would shift peaks in time.
applyForward(signal, coeffs);
std::reverse(signal.begin(), signal.end());
applyForward(signal, coeffs);
std::reverse(signal.begin(), signal.end());

// Bad — restates what code already says
// Reverse the signal
std::reverse(signal.begin(), signal.end());
```

### 6.5 TODO Comments

TODOs are forbidden in production code. Use Jira tickets instead.

During development, if you must add a TODO:

```cpp
// TODO(ONYX-123): Optimise for large windows using FFT convolution
```

All TODOs must be resolved before merge to main.

---

## 7. Error Handling

### 7.1 Strategy

OnYX uses a **fail-fast** strategy for programming errors and a **graceful degradation** strategy for runtime errors.

| Error Type | Strategy | Mechanism |
|------------|----------|-----------|
| Programming errors (bugs) | Fail fast | Assertions, exceptions |
| Invalid user input | Reject with message | Return codes, user feedback |
| Resource failures | Graceful degradation | Result type, fallbacks |
| Recoverable runtime errors | Handle and continue | Result type |
| Unrecoverable errors | Clean shutdown | Exception, error callback |

### 7.2 Result Type

Use `Result<T>` for operations that can fail at runtime:

```cpp
template<typename T>
class Result
{
public:
    static Result success(T value);
    static Result failure(std::string error);

    bool ok() const;
    const T& value() const;       // Throws if not ok
    const std::string& error() const;

    // Monadic operations
    template<typename F>
    auto map(F&& f) -> Result<decltype(f(std::declval<T>()))>;

    template<typename F>
    auto flatMap(F&& f) -> decltype(f(std::declval<T>()));
};
```

Usage:

```cpp
Result<MotionPath> analyseClip(const FrameSequence& frames)
{
    if (frames.empty())
    {
        return Result<MotionPath>::failure("Cannot analyse empty frame sequence");
    }

    auto gpuResult = allocateGpuBuffers(frames.width(), frames.height());
    if (!gpuResult.ok())
    {
        return Result<MotionPath>::failure("GPU allocation failed: " + gpuResult.error());
    }

    // ... proceed with analysis ...

    return Result<MotionPath>::success(std::move(path));
}
```

### 7.3 CUDA Error Handling

All CUDA calls must be wrapped with error checking:

```cpp
// In CudaError.h
#define CUDA_CHECK(call)                                                    \
    do {                                                                    \
        cudaError_t err = (call);                                          \
        if (err != cudaSuccess)                                            \
        {                                                                   \
            throw CudaException(__FILE__, __LINE__, #call,                 \
                               cudaGetErrorString(err));                   \
        }                                                                   \
    } while (0)

#define CUFFT_CHECK(call)                                                   \
    do {                                                                    \
        cufftResult err = (call);                                          \
        if (err != CUFFT_SUCCESS)                                          \
        {                                                                   \
            throw CufftException(__FILE__, __LINE__, #call, err);          \
        }                                                                   \
    } while (0)
```

Usage:

```cpp
void PhaseCorrelator::allocate(int width, int height)
{
    size_t size = width * height * sizeof(float);
    CUDA_CHECK(cudaMalloc(&deviceBuffer_, size));
    CUFFT_CHECK(cufftPlan2d(&fftPlan_, height, width, CUFFT_R2C));
}
```

### 7.4 Assertions

Use assertions for invariants that should never be violated:

```cpp
void ButterworthFilter::applyForward(std::vector<double>& signal,
                                      const BiquadCoeffs& coeffs) const
{
    assert(!signal.empty() && "Signal must not be empty");
    assert(std::isfinite(coeffs.b0) && "Coefficients must be finite");

    // ... implementation
}
```

Assertions are **disabled in Release builds**. Never use assertions for:
- User input validation
- Resource allocation
- Anything that can legitimately fail at runtime

### 7.5 Defensive Numerical Programming

Guard against numerical issues:

```cpp
double safeDivide(double numerator, double denominator, double fallback = 0.0)
{
    if (std::abs(denominator) < std::numeric_limits<double>::epsilon())
    {
        return fallback;
    }
    return numerator / denominator;
}

bool isValidFloat(float value)
{
    return std::isfinite(value);
}

float clampToValid(float value, float minVal, float maxVal)
{
    if (!std::isfinite(value)) return (minVal + maxVal) / 2.0f;
    return std::clamp(value, minVal, maxVal);
}
```

---

## 8. Memory Management

### 8.1 Ownership Model

- **Single ownership** by default: `std::unique_ptr<T>`
- **Shared ownership** only when genuinely needed: `std::shared_ptr<T>`
- **Non-owning references**: raw pointers or references
- **Never** use raw `new`/`delete` for ownership

```cpp
// Good — clear ownership
class StabilisationPipeline
{
public:
    StabilisationPipeline(std::unique_ptr<IAnalyser> analyser)
        : analyser_(std::move(analyser)) {}

    void setCallback(IProgressCallback* callback)  // Non-owning
    {
        callback_ = callback;
    }

private:
    std::unique_ptr<IAnalyser> analyser_;  // Owns
    IProgressCallback* callback_ = nullptr; // Does not own
};

// Bad — unclear ownership
class StabilisationPipeline
{
public:
    void setAnalyser(IAnalyser* analyser)  // Who owns this?
    {
        analyser_ = analyser;  // Leak if previously set? Double-free?
    }
};
```

### 8.2 RAII for Resources

All resources use RAII wrappers:

```cpp
// CUDA memory RAII wrapper
template<typename T>
class CudaBuffer
{
public:
    CudaBuffer() : ptr_(nullptr), size_(0) {}

    explicit CudaBuffer(size_t count)
        : size_(count)
    {
        CUDA_CHECK(cudaMalloc(&ptr_, count * sizeof(T)));
    }

    ~CudaBuffer()
    {
        if (ptr_)
        {
            cudaFree(ptr_);  // Don't throw in destructor
        }
    }

    // Move only, no copy
    CudaBuffer(CudaBuffer&& other) noexcept
        : ptr_(other.ptr_), size_(other.size_)
    {
        other.ptr_ = nullptr;
        other.size_ = 0;
    }

    CudaBuffer& operator=(CudaBuffer&& other) noexcept
    {
        if (this != &other)
        {
            if (ptr_) cudaFree(ptr_);
            ptr_ = other.ptr_;
            size_ = other.size_;
            other.ptr_ = nullptr;
            other.size_ = 0;
        }
        return *this;
    }

    CudaBuffer(const CudaBuffer&) = delete;
    CudaBuffer& operator=(const CudaBuffer&) = delete;

    T* get() { return ptr_; }
    const T* get() const { return ptr_; }
    size_t size() const { return size_; }

private:
    T* ptr_;
    size_t size_;
};
```

### 8.3 Container Preferences

| Need | Use |
|------|-----|
| Dynamic array | `std::vector<T>` |
| Fixed-size array | `std::array<T, N>` |
| Key-value lookup | `std::unordered_map<K, V>` |
| Ordered key-value | `std::map<K, V>` |
| String | `std::string` |
| Optional value | `std::optional<T>` |
| Type-safe union | `std::variant<T...>` |

Pre-allocate when size is known:

```cpp
// Good
std::vector<double> velocities;
velocities.reserve(frameCount);  // Avoid reallocations

// Bad
std::vector<double> velocities;
for (int i = 0; i < frameCount; ++i)
{
    velocities.push_back(computeVelocity(i));  // May reallocate many times
}
```

---

## 9. CUDA Conventions

### 9.1 Kernel Naming and Organisation

```cpp
// In Kernels.h — declarations only
namespace onyx
{
namespace cuda
{

void launchRenderKernel(
    const float* src, float* dst,
    int width, int height,
    float cosAngle, float sinAngle,
    float dx, float dy,
    float scale,
    float borderR, float borderG, float borderB,
    cudaStream_t stream = nullptr
);

void launchLumaKernel(
    const float* rgba, float* luma,
    int width, int height,
    cudaStream_t stream = nullptr
);

} // namespace cuda
} // namespace onyx
```

### 9.2 Kernel Implementation

```cpp
// In Kernels.cu
namespace
{

__global__ void RenderKernel(
    const float4* __restrict__ src,
    float4* __restrict__ dst,
    int width, int height,
    float cosA, float sinA,
    float dx, float dy,
    float scale,
    float borderR, float borderG, float borderB)
{
    const int x = blockIdx.x * blockDim.x + threadIdx.x;
    const int y = blockIdx.y * blockDim.y + threadIdx.y;

    if (x >= width || y >= height) return;

    // ... kernel body ...
}

} // anonymous namespace

namespace onyx
{
namespace cuda
{

void launchRenderKernel(
    const float* src, float* dst,
    int width, int height,
    float cosAngle, float sinAngle,
    float dx, float dy,
    float scale,
    float borderR, float borderG, float borderB,
    cudaStream_t stream)
{
    const dim3 blockSize(16, 16);
    const dim3 gridSize(
        (width + blockSize.x - 1) / blockSize.x,
        (height + blockSize.y - 1) / blockSize.y
    );

    RenderKernel<<<gridSize, blockSize, 0, stream>>>(
        reinterpret_cast<const float4*>(src),
        reinterpret_cast<float4*>(dst),
        width, height,
        cosAngle, sinAngle,
        dx, dy,
        scale,
        borderR, borderG, borderB
    );

    CUDA_CHECK(cudaGetLastError());
}

} // namespace cuda
} // namespace onyx
```

### 9.3 CUDA Best Practices

1. **Always check errors** after kernel launches
2. **Use streams** for async operations
3. **Prefer `__restrict__`** to hint non-aliasing
4. **Coalesce memory access** — adjacent threads access adjacent memory
5. **Avoid warp divergence** — minimise conditionals
6. **Document thread/block assumptions** in kernel comments

---

## 10. Testing Standards

### 10.1 Test Structure

Follow Arrange-Act-Assert pattern:

```cpp
TEST(ButterworthFilterTest, DesignLowpassReturnsValidCoefficients)
{
    // Arrange
    ButterworthFilter filter(48000.0);
    const double cutoffHz = 1.0;

    // Act
    BiquadCoeffs coeffs = filter.designLowpass(cutoffHz);

    // Assert
    EXPECT_TRUE(std::isfinite(coeffs.b0));
    EXPECT_TRUE(std::isfinite(coeffs.a1));
    EXPECT_GT(coeffs.b0, 0.0);
}
```

### 10.2 Test Naming

`MethodName_Scenario_ExpectedBehavior`

```cpp
TEST(ButterworthFilterTest, DesignLowpass_CutoffAtNyquist_ThrowsInvalidArgument)
TEST(ButterworthFilterTest, ApplyZeroPhase_EmptySignal_ReturnsImmediately)
TEST(ButterworthFilterTest, ApplyZeroPhase_SineWave_PreservesPhase)
```

### 10.3 Test Coverage

Every public method must have tests covering:
- Normal operation (happy path)
- Edge cases (empty input, boundary values)
- Error conditions (invalid input, resource failures)

---

## 11. Version Control

### 11.1 Branch Naming

| Type | Format | Example |
|------|--------|---------|
| Feature | `feature/ONYX-123-description` | `feature/ONYX-15-linux-cmake` |
| Bugfix | `fix/ONYX-123-description` | `fix/ONYX-99-null-pointer` |
| Hotfix | `hotfix/ONYX-123-description` | `hotfix/ONYX-101-crash-on-load` |

### 11.2 Commit Messages

Follow Conventional Commits:

```
type(scope): subject

body

footer
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `build`, `ci`

```
feat(core): Add Butterworth zero-phase filter

Implements forward-backward filtering to achieve zero phase distortion.
Uses bilinear transform for coefficient calculation.

Closes ONYX-39
```

### 11.3 Pull Request Requirements

Before merging:
- [ ] All CI checks pass
- [ ] Code review approved
- [ ] Jira ticket linked
- [ ] No unresolved TODOs
- [ ] Documentation updated (if public API changed)
- [ ] Test coverage maintained or improved

---

## 12. Code Review Checklist

Reviewers verify:

### Correctness
- [ ] Logic is correct
- [ ] Edge cases handled
- [ ] No off-by-one errors
- [ ] Thread safety (if applicable)

### SOLID Compliance
- [ ] Single responsibility maintained
- [ ] Depends on abstractions, not concretions
- [ ] No unexpected side effects
- [ ] Interfaces are minimal

### Quality
- [ ] Naming is clear and consistent
- [ ] No code duplication
- [ ] No magic numbers
- [ ] Comments explain "why", not "what"

### Safety
- [ ] All resources properly managed (RAII)
- [ ] All errors handled appropriately
- [ ] No memory leaks possible
- [ ] No undefined behaviour

### Performance (where relevant)
- [ ] No unnecessary allocations
- [ ] Appropriate algorithm complexity
- [ ] CUDA kernels follow best practices

---

## Appendix A: Clang-Format Configuration

Save as `.clang-format` in repository root:

```yaml
---
Language: Cpp
BasedOnStyle: LLVM
IndentWidth: 4
TabWidth: 4
UseTab: Never
BreakBeforeBraces: Allman
AllowShortFunctionsOnASingleLine: None
AllowShortIfStatementsOnASingleLine: Never
AllowShortLoopsOnASingleLine: false
ColumnLimit: 100
NamespaceIndentation: All
FixNamespaceComments: true
SortIncludes: true
IncludeBlocks: Regroup
IncludeCategories:
  - Regex: '^".*\.h"'
    Priority: 1
  - Regex: '^<c.*>'
    Priority: 2
  - Regex: '^<.*>'
    Priority: 3
  - Regex: '.*'
    Priority: 4
---
```

---

## Appendix B: Clang-Tidy Configuration

Save as `.clang-tidy` in repository root:

```yaml
---
Checks: >
  *,
  -fuchsia-*,
  -google-*,
  -llvm-*,
  -llvmlibc-*,
  -altera-*,
  -abseil-*,
  -android-*,
  -zircon-*,
  -readability-magic-numbers,
  -cppcoreguidelines-avoid-magic-numbers,
  -modernize-use-trailing-return-type
WarningsAsErrors: ''
HeaderFilterRegex: '.*'
CheckOptions:
  - key: readability-identifier-naming.ClassCase
    value: CamelCase
  - key: readability-identifier-naming.FunctionCase
    value: camelBack
  - key: readability-identifier-naming.VariableCase
    value: camelBack
  - key: readability-identifier-naming.PrivateMemberSuffix
    value: '_'
  - key: readability-identifier-naming.ConstantCase
    value: UPPER_CASE
  - key: readability-identifier-naming.NamespaceCase
    value: lower_case
---
```

---

**Document Control**

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02 | S. Baker / Claude | Initial release |

---

*This document is the authoritative reference for OnYX code standards. All production code must comply. Deviations require documented justification and approval.*

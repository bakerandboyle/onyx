# OnYX Dependency Manifest

**Version:** 1.0  
**Last Updated:** February 2026  
**Maintainer:** S. Baker

---

## Overview

This document lists all external dependencies for the OnYX video stabilisation plugin. All dependencies must be reviewed for license compatibility before inclusion.

**License Compatibility:** OnYX is proprietary software. Dependencies must permit use in proprietary products.

---

## 1. Build-Time Dependencies

### 1.1 CUDA Toolkit

| Attribute | Value |
|-----------|-------|
| **Name** | NVIDIA CUDA Toolkit |
| **Version (Minimum)** | 11.0 |
| **Version (Tested)** | 11.8, 12.0, 12.3 |
| **Version (Recommended)** | 12.x (latest stable) |
| **Source** | https://developer.nvidia.com/cuda-toolkit |
| **License** | NVIDIA CUDA Toolkit EULA |
| **License Type** | Proprietary (permits redistribution of runtime) |
| **Components Used** | cudart, cuFFT, nvcc compiler |
| **Required** | Yes |
| **Platforms** | Linux, Windows (macOS deprecated by NVIDIA) |

**License Notes:**
- CUDA runtime libraries may be redistributed with applications
- Must include NVIDIA copyright notice
- Cannot modify CUDA libraries
- See: https://docs.nvidia.com/cuda/eula/index.html

**Bundling Strategy:**
- Installer checks for CUDA runtime presence
- Provide download link if not present
- Do NOT bundle CUDA runtime (user installs separately)

---

### 1.2 OpenFX SDK

| Attribute | Value |
|-----------|-------|
| **Name** | OpenFX (OFX) Image Processing API |
| **Version (Minimum)** | 1.4 |
| **Version (Tested)** | 1.4 |
| **Source** | https://github.com/AcademySoftwareFoundation/openfx |
| **License** | BSD 3-Clause |
| **License Type** | Permissive |
| **Components Used** | ofxCore.h, ofxImageEffect.h, Support library |
| **Required** | Yes |
| **Platforms** | Linux, macOS, Windows |

**License Notes:**
- Permissive license allows proprietary use
- Must retain copyright notice in source files
- No copyleft/viral obligations

**Full License Text:**
```
Copyright (c) Contributors to the OpenFX Project. All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

1. Redistributions of source code must retain the above copyright notice,
   this list of conditions and the following disclaimer.

2. Redistributions in binary form must reproduce the above copyright notice,
   this list of conditions and the following disclaimer in the documentation
   and/or other materials provided with the distribution.

3. Neither the name of the copyright holder nor the names of its contributors
   may be used to endorse or promote products derived from this software
   without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE
LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
CONSEQUENTIAL DAMAGES.
```

---

### 1.3 C++ Standard Library

| Attribute | Value |
|-----------|-------|
| **Name** | C++ Standard Library |
| **Standard** | C++17 |
| **Source** | Provided by compiler (GCC, Clang, MSVC) |
| **License** | Varies by implementation |
| **Components Used** | STL containers, algorithms, threading, filesystem |
| **Required** | Yes |
| **Platforms** | All |

**Implementation-Specific:**
| Platform | Implementation | License |
|----------|----------------|---------|
| Linux (GCC) | libstdc++ | GPL with Runtime Exception |
| Linux (Clang) | libc++ | Apache 2.0 with LLVM Exception |
| macOS | libc++ | Apache 2.0 with LLVM Exception |
| Windows | MSVC STL | Microsoft license (permits redistribution) |

**License Notes:**
- GCC Runtime Exception permits proprietary linking
- No source disclosure required

---

### 1.4 CMake

| Attribute | Value |
|-----------|-------|
| **Name** | CMake |
| **Version (Minimum)** | 3.18 |
| **Version (Recommended)** | 3.25+ |
| **Source** | https://cmake.org/ |
| **License** | BSD 3-Clause |
| **License Type** | Permissive |
| **Components Used** | Build system only (not distributed) |
| **Required** | Yes (build only) |
| **Platforms** | All |

**License Notes:**
- Build tool only, not linked or distributed
- No license obligations for output binaries

---

## 2. Runtime Dependencies

### 2.1 CUDA Runtime Library (cudart)

| Attribute | Value |
|-----------|-------|
| **Name** | CUDA Runtime Library |
| **Provided By** | CUDA Toolkit installation |
| **License** | NVIDIA CUDA Toolkit EULA |
| **Required** | Yes |
| **User Must Install** | Yes (CUDA Toolkit or driver package) |

**Minimum Compute Capability:** 5.0 (Maxwell)  
**Recommended Compute Capability:** 7.0+ (Volta/Turing/Ampere)

---

### 2.2 cuFFT Library

| Attribute | Value |
|-----------|-------|
| **Name** | NVIDIA cuFFT |
| **Provided By** | CUDA Toolkit installation |
| **License** | NVIDIA CUDA Toolkit EULA |
| **Required** | Yes |
| **Purpose** | Fast Fourier Transform for phase correlation |

---

### 2.3 DaVinci Resolve (Host Application)

| Attribute | Value |
|-----------|-------|
| **Name** | DaVinci Resolve |
| **Version (Minimum)** | 17.0 |
| **Version (Tested)** | 17.4, 18.0, 18.5, 19.0 |
| **Editions** | Free and Studio |
| **Vendor** | Blackmagic Design |
| **Required** | Yes (host application) |

**Notes:**
- OnYX is an OFX plugin, requires OFX-compatible host
- Primary target is DaVinci Resolve
- May work in other OFX hosts (Nuke, Fusion standalone) — untested

---

## 3. Optional Dependencies

### 3.1 GoogleTest (Testing Only)

| Attribute | Value |
|-----------|-------|
| **Name** | GoogleTest |
| **Version** | 1.14.0 |
| **Source** | https://github.com/google/googletest |
| **License** | BSD 3-Clause |
| **License Type** | Permissive |
| **Required** | No (testing only) |
| **Distributed** | No |

**Acquisition:**
```cmake
include(FetchContent)
FetchContent_Declare(
  googletest
  GIT_REPOSITORY https://github.com/google/googletest.git
  GIT_TAG v1.14.0
)
FetchContent_MakeAvailable(googletest)
```

---

### 3.2 cppcheck (Static Analysis Only)

| Attribute | Value |
|-----------|-------|
| **Name** | cppcheck |
| **Version** | 2.10+ |
| **Source** | https://cppcheck.sourceforge.io/ |
| **License** | GPL v3 |
| **Required** | No (development only) |
| **Distributed** | No |

**Notes:**
- GPL does not apply as tool is not linked or distributed
- Used for static analysis during development only

---

### 3.3 clang-tidy (Static Analysis Only)

| Attribute | Value |
|-----------|-------|
| **Name** | clang-tidy |
| **Version** | 15.0+ |
| **Source** | https://clang.llvm.org/extra/clang-tidy/ |
| **License** | Apache 2.0 with LLVM Exception |
| **Required** | No (development only) |
| **Distributed** | No |

---

### 3.4 clang-format (Development Only)

| Attribute | Value |
|-----------|-------|
| **Name** | clang-format |
| **Version** | 15.0+ |
| **Source** | https://clang.llvm.org/docs/ClangFormat.html |
| **License** | Apache 2.0 with LLVM Exception |
| **Required** | No (development only) |
| **Distributed** | No |

---

## 4. Considered but Rejected

| Library | Purpose | Reason Rejected |
|---------|---------|-----------------|
| OpenCV | Image processing | Too heavy, CUDA provides what we need |
| Eigen | Linear algebra | Not needed for current algorithms |
| FFmpeg | Video I/O | Host application handles I/O |
| Qt | GUI | OFX provides parameter UI |
| Boost | Utilities | C++17 standard library sufficient |

---

## 5. Future Considerations

### 5.1 macOS Apple Silicon Support

Apple Silicon Macs do not support CUDA. Options:

| Option | Status | Notes |
|--------|--------|-------|
| Metal Compute | Investigate | Would require kernel rewrites |
| OpenCL | Deprecated | Apple deprecated OpenCL |
| CPU Fallback | Possible | Significant performance reduction |
| No Support | Current | Document as Windows/Linux only |

**Decision:** Defer Apple Silicon support. Current market focus is NVIDIA GPU users.

### 5.2 Potential Future Dependencies

| Library | Purpose | When Needed |
|---------|---------|-------------|
| libcurl | License validation | If online activation required |
| nlohmann/json | Config files | If JSON config needed |
| spdlog | Logging | If structured logging needed |

---

## 6. License Compliance Checklist

### 6.1 Notices Required in Distribution

| Dependency | Notice Required | Location |
|------------|-----------------|----------|
| OpenFX | Yes - BSD notice | THIRD_PARTY_LICENSES.txt |
| CUDA | Yes - NVIDIA notice | THIRD_PARTY_LICENSES.txt |
| GoogleTest | No (not distributed) | N/A |

### 6.2 THIRD_PARTY_LICENSES.txt Template

```
OnYX Video Stabilisation Plugin
Third-Party Software Notices

================================================================================
OpenFX SDK
================================================================================
Copyright (c) Contributors to the OpenFX Project. All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:
[... full BSD 3-Clause text ...]

================================================================================
NVIDIA CUDA
================================================================================
This software uses NVIDIA CUDA Toolkit libraries.
Copyright (c) 2005-2026 NVIDIA Corporation. All rights reserved.

CUDA is a trademark of NVIDIA Corporation.

Portions of this software are licensed under the NVIDIA CUDA Toolkit 
End User License Agreement. See:
https://docs.nvidia.com/cuda/eula/index.html
================================================================================
```

---

## 7. Verification Commands

### 7.1 Check CUDA Version

```bash
nvcc --version
nvidia-smi
```

### 7.2 Check CMake Version

```bash
cmake --version
```

### 7.3 Verify OFX Installation

```bash
# Linux
ls /usr/OFX/Plugins/

# macOS
ls /Library/OFX/Plugins/

# Windows
dir "C:\Program Files\Common Files\OFX\Plugins"
```

---

## 8. Version Lock File

For reproducible builds, exact tested versions:

```yaml
# onyx-dependencies.lock.yml
dependencies:
  cuda_toolkit:
    version: "12.3.1"
    sha256: "abc123..."  # installer checksum
  
  openfx:
    version: "1.4"
    git_commit: "a1b2c3d4..."
    repository: "https://github.com/AcademySoftwareFoundation/openfx"
  
  googletest:
    version: "1.14.0"
    git_tag: "v1.14.0"
  
build_tools:
  cmake:
    minimum: "3.18"
    tested: "3.28.1"
  
  gcc:
    minimum: "9.0"
    tested: "13.2.0"
  
  msvc:
    minimum: "19.29"  # VS 2019 16.10
    tested: "19.38"   # VS 2022 17.8

platforms_tested:
  linux:
    - "Ubuntu 22.04 LTS"
    - "Ubuntu 24.04 LTS"
  windows:
    - "Windows 10 22H2"
    - "Windows 11 23H2"
  macos:
    - "macOS 13 Ventura (Intel only)"
    - "macOS 14 Sonoma (Intel only)"
```

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02 | S. Baker | Initial release |

---

*This document must be updated whenever dependencies change. All new dependencies require license review before adoption.*

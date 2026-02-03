# Building OnYX

## Prerequisites

### Linux (Ubuntu 22.04 / 24.04)

#### Required

1. **C++17 Compiler**
   ```bash
   sudo apt update
   sudo apt install build-essential g++
   ```

2. **CMake 3.18+**
   ```bash
   sudo apt install cmake
   cmake --version  # Verify 3.18+
   ```

3. **CUDA Toolkit 11.0+**
   
   Download from NVIDIA: https://developer.nvidia.com/cuda-downloads
   
   Or install via apt (Ubuntu):
   ```bash
   # Add NVIDIA repository (if not already added)
   wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
   sudo dpkg -i cuda-keyring_1.1-1_all.deb
   sudo apt update
   
   # Install CUDA Toolkit
   sudo apt install cuda-toolkit-12-3
   
   # Add to PATH (add to ~/.bashrc)
   export PATH=/usr/local/cuda/bin:$PATH
   export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
   ```
   
   Verify:
   ```bash
   nvcc --version
   nvidia-smi
   ```

4. **Git**
   ```bash
   sudo apt install git
   ```

#### Optional (for development)

```bash
# Static analysis tools
sudo apt install clang-tidy cppcheck

# Code formatting
sudo apt install clang-format

# Documentation
sudo apt install doxygen graphviz
```

### OpenFX SDK

Clone into the external directory:
```bash
cd /path/to/onyx
git clone https://github.com/AcademySoftwareFoundation/openfx.git external/openfx
```

---

## Building

### Quick Start

```bash
# Clone repository
git clone https://github.com/bakerandboyle/onyx.git
cd onyx

# Get OpenFX SDK
git clone https://github.com/AcademySoftwareFoundation/openfx.git external/openfx

# Create build directory
mkdir build && cd build

# Configure (Release build)
cmake .. -DCMAKE_BUILD_TYPE=Release

# Build (parallel)
cmake --build . --parallel $(nproc)
```

### Build Types

| Type | Use Case | Flags |
|------|----------|-------|
| `Debug` | Development, debugging | `-O0 -g -DDEBUG` |
| `Release` | Production | `-O3 -DNDEBUG` |
| `RelWithDebInfo` | Profiling | `-O2 -g -DNDEBUG` |

```bash
cmake .. -DCMAKE_BUILD_TYPE=Debug
cmake .. -DCMAKE_BUILD_TYPE=Release
cmake .. -DCMAKE_BUILD_TYPE=RelWithDebInfo
```

### Build Options

| Option | Default | Description |
|--------|---------|-------------|
| `ONYX_BUILD_TESTS` | ON | Build unit tests |
| `ONYX_ENABLE_STATIC_ANALYSIS` | OFF | Enable clang-tidy |
| `ONYX_CUDA_ARCH` | auto | CUDA architectures |

```bash
# Build without tests
cmake .. -DONYX_BUILD_TESTS=OFF

# Enable static analysis
cmake .. -DONYX_ENABLE_STATIC_ANALYSIS=ON

# Target specific GPU architectures
cmake .. -DONYX_CUDA_ARCH="75;86"
```

### CUDA Architecture Reference

| Architecture | GPUs | Compute Capability |
|--------------|------|-------------------|
| Maxwell | GTX 9xx | 50, 52 |
| Pascal | GTX 10xx | 60, 61 |
| Volta | Titan V | 70 |
| Turing | RTX 20xx | 75 |
| Ampere | RTX 30xx | 80, 86 |
| Ada Lovelace | RTX 40xx | 89 |
| Hopper | H100 | 90 |

Default builds target 60+ (Pascal and newer).

---

## Running Tests

```bash
cd build

# Run all tests
ctest --output-on-failure

# Or directly
./onyx_tests

# Verbose output
ctest -V
```

---

## Installation

### User Installation (Recommended)

Installs to `~/.OFX/Plugins/` — no sudo required:

```bash
cmake --build . --target install-user
```

Verify:
```bash
ls -la ~/.OFX/Plugins/OnYX.ofx.bundle/
```

### System Installation

Installs to `/usr/OFX/Plugins/` — requires sudo:

```bash
sudo cmake --install .
```

Verify:
```bash
ls -la /usr/OFX/Plugins/OnYX.ofx.bundle/
```

### Bundle Structure

After build, the plugin bundle is at:
```
build/OnYX.ofx.bundle/
├── Contents/
│   ├── Info.xml
│   ├── Linux-x86-64/
│   │   └── OnYX.ofx
│   └── Resources/
│       └── THIRD_PARTY_LICENSES.txt
```

---

## Verification

### Check Plugin Loads in DaVinci Resolve

1. Launch DaVinci Resolve
2. Open a project
3. Go to Color page or Edit page
4. Open Effects Library
5. Search for "OnYX"
6. If found, plugin is correctly installed

### Troubleshooting

**Plugin not appearing:**
```bash
# Check bundle exists
ls ~/.OFX/Plugins/OnYX.ofx.bundle/Contents/Linux-x86-64/OnYX.ofx

# Check library dependencies
ldd ~/.OFX/Plugins/OnYX.ofx.bundle/Contents/Linux-x86-64/OnYX.ofx

# Look for missing CUDA libraries
ldd ~/.OFX/Plugins/OnYX.ofx.bundle/Contents/Linux-x86-64/OnYX.ofx | grep "not found"
```

**CUDA errors:**
```bash
# Verify CUDA installation
nvidia-smi
nvcc --version

# Check GPU compute capability
nvidia-smi --query-gpu=compute_cap --format=csv
```

---

## Development Workflow

### Format Code

```bash
# Format all source files
find src tests -name "*.cpp" -o -name "*.h" -o -name "*.cu" | xargs clang-format -i
```

### Static Analysis

```bash
# Build with analysis enabled
cmake .. -DONYX_ENABLE_STATIC_ANALYSIS=ON
cmake --build .

# Or run manually
cppcheck --enable=all --std=c++17 src/
```

### Clean Build

```bash
rm -rf build
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
cmake --build . --parallel
```

---

## IDE Setup

### VS Code

Recommended extensions:
- C/C++ (Microsoft)
- CMake Tools
- clangd

Settings (`.vscode/settings.json`):
```json
{
    "cmake.buildDirectory": "${workspaceFolder}/build",
    "cmake.configureOnOpen": true,
    "C_Cpp.default.configurationProvider": "ms-vscode.cmake-tools"
}
```

### CLion

1. Open project folder
2. CLion auto-detects CMakeLists.txt
3. Configure toolchain with CUDA compiler
4. Build/Run as normal

---

## Continuous Integration

The CI pipeline runs on every push:

1. **Build** — Linux (Ubuntu 22.04, GCC 11)
2. **Test** — Unit tests with GoogleTest
3. **Analysis** — Static analysis (optional)

See `.github/workflows/` for CI configuration.

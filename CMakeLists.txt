# ==============================================================================
# OnYX Video Stabilisation Plugin
# CMake Build Configuration
# ==============================================================================
#
# Copyright (c) 2026 Baker & Boyle Research
# License: Proprietary
#
# Usage:
#   mkdir build && cd build
#   cmake .. -DCMAKE_BUILD_TYPE=Release
#   cmake --build . --parallel
#
# Options:
#   -DCMAKE_BUILD_TYPE=Debug|Release|RelWithDebInfo
#   -DONYX_BUILD_TESTS=ON|OFF (default: ON)
#   -DONYX_ENABLE_STATIC_ANALYSIS=ON|OFF (default: OFF)
#   -DONYX_CUDA_ARCH="75;86" (override auto-detection)
#
# ==============================================================================

cmake_minimum_required(VERSION 3.18 FATAL_ERROR)

# ------------------------------------------------------------------------------
# Project Definition
# ------------------------------------------------------------------------------

project(OnYX
    VERSION 1.0.0
    DESCRIPTION "GPU-accelerated video stabilisation plugin for DaVinci Resolve"
    HOMEPAGE_URL "https://bakerandboyle.com/onyx"
    LANGUAGES CXX CUDA
)

# Prevent in-source builds
if(CMAKE_SOURCE_DIR STREQUAL CMAKE_BINARY_DIR)
    message(FATAL_ERROR "In-source builds are not allowed. Use: mkdir build && cd build && cmake ..")
endif()

# ------------------------------------------------------------------------------
# Build Options
# ------------------------------------------------------------------------------

option(ONYX_BUILD_TESTS "Build unit tests" ON)
option(ONYX_ENABLE_STATIC_ANALYSIS "Enable clang-tidy static analysis" OFF)
option(ONYX_INSTALL_DEV "Install development files (headers)" OFF)

set(ONYX_CUDA_ARCH "" CACHE STRING "CUDA architectures (e.g., '75;86'). Empty = auto-detect.")

# ------------------------------------------------------------------------------
# Global Settings
# ------------------------------------------------------------------------------

# C++ Standard
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# CUDA Standard
set(CMAKE_CUDA_STANDARD 17)
set(CMAKE_CUDA_STANDARD_REQUIRED ON)

# Position Independent Code (required for shared libraries)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# Export compile commands for IDE support
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Default build type
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    message(STATUS "Setting build type to 'Release' as none was specified.")
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "RelWithDebInfo" "MinSizeRel")
endif()

# ------------------------------------------------------------------------------
# Version Information
# ------------------------------------------------------------------------------

find_package(Git QUIET)
if(GIT_FOUND)
    execute_process(
        COMMAND ${GIT_EXECUTABLE} rev-parse --short HEAD
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE ONYX_GIT_HASH
        OUTPUT_STRIP_TRAILING_WHITESPACE
        ERROR_QUIET
    )
    execute_process(
        COMMAND ${GIT_EXECUTABLE} describe --tags --always --dirty
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE ONYX_GIT_DESCRIBE
        OUTPUT_STRIP_TRAILING_WHITESPACE
        ERROR_QUIET
    )
else()
    set(ONYX_GIT_HASH "unknown")
    set(ONYX_GIT_DESCRIBE "unknown")
endif()

string(TIMESTAMP ONYX_BUILD_DATE "%Y-%m-%d")
string(TIMESTAMP ONYX_BUILD_TIME "%H:%M:%S")

message(STATUS "OnYX Version: ${PROJECT_VERSION}")
message(STATUS "Git: ${ONYX_GIT_DESCRIBE}")

# ------------------------------------------------------------------------------
# Find Dependencies
# ------------------------------------------------------------------------------

# CUDA Toolkit
find_package(CUDAToolkit REQUIRED)
message(STATUS "CUDA Toolkit: ${CUDAToolkit_VERSION}")

if(CUDAToolkit_VERSION VERSION_LESS "11.0")
    message(FATAL_ERROR "CUDA Toolkit 11.0+ required. Found: ${CUDAToolkit_VERSION}")
endif()

if(NOT TARGET CUDA::cufft)
    message(FATAL_ERROR "cuFFT not found. Ensure CUDA Toolkit is properly installed.")
endif()

# OpenFX SDK
find_path(OPENFX_INCLUDE_DIR
    NAMES ofxCore.h
    PATHS
        ${CMAKE_SOURCE_DIR}/external/openfx/include
        ${CMAKE_SOURCE_DIR}/third_party/openfx/include
        /usr/include/openfx
        /usr/local/include/openfx
    PATH_SUFFIXES openfx
    DOC "OpenFX SDK include directory"
)

if(NOT OPENFX_INCLUDE_DIR)
    message(FATAL_ERROR 
        "OpenFX SDK not found. Clone into external/openfx:\n"
        "  git clone https://github.com/AcademySoftwareFoundation/openfx.git external/openfx"
    )
endif()

message(STATUS "OpenFX SDK: ${OPENFX_INCLUDE_DIR}")

# Check for OpenFX Support library (optional but useful)
find_path(OPENFX_SUPPORT_INCLUDE_DIR
    NAMES ofxsImageEffect.h
    PATHS
        ${CMAKE_SOURCE_DIR}/external/openfx/Support/include
        /usr/include/openfx/Support
    DOC "OpenFX Support library include directory"
)

# Threads
find_package(Threads REQUIRED)

# ------------------------------------------------------------------------------
# CUDA Architecture Configuration
# ------------------------------------------------------------------------------

if(ONYX_CUDA_ARCH)
    set(CMAKE_CUDA_ARCHITECTURES ${ONYX_CUDA_ARCH})
    message(STATUS "CUDA Architectures (user): ${CMAKE_CUDA_ARCHITECTURES}")
else()
    # Sensible defaults based on CUDA version
    if(CUDAToolkit_VERSION VERSION_GREATER_EQUAL "12.0")
        set(CMAKE_CUDA_ARCHITECTURES "60;70;75;80;86;89;90")
    elseif(CUDAToolkit_VERSION VERSION_GREATER_EQUAL "11.0")
        set(CMAKE_CUDA_ARCHITECTURES "60;70;75;80;86")
    else()
        set(CMAKE_CUDA_ARCHITECTURES "50;60;70;75")
    endif()
    message(STATUS "CUDA Architectures (auto): ${CMAKE_CUDA_ARCHITECTURES}")
endif()

# ------------------------------------------------------------------------------
# Compiler Warnings
# ------------------------------------------------------------------------------

# Interface library for compile options
add_library(onyx_warnings INTERFACE)

if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    target_compile_options(onyx_warnings INTERFACE
        $<$<COMPILE_LANGUAGE:CXX>:
            -Wall
            -Wextra
            -Wpedantic
            -Wconversion
            -Wsign-conversion
            -Wcast-qual
            -Wformat=2
            -Wundef
            -Werror=return-type
            -Wno-unused-parameter
        >
    )
    
    # GCC-specific
    if(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
        target_compile_options(onyx_warnings INTERFACE
            $<$<COMPILE_LANGUAGE:CXX>:
                -Wlogical-op
                -Wduplicated-cond
            >
        )
    endif()
    
    # Clang-specific
    if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
        target_compile_options(onyx_warnings INTERFACE
            $<$<COMPILE_LANGUAGE:CXX>:
                -Wshadow-all
            >
        )
    endif()
endif()

# CUDA compiler options
add_library(onyx_cuda_options INTERFACE)
target_compile_options(onyx_cuda_options INTERFACE
    $<$<COMPILE_LANGUAGE:CUDA>:
        --expt-relaxed-constexpr
        -Xcompiler=-Wall
        $<$<CONFIG:Debug>:-G>
        $<$<CONFIG:Debug>:-lineinfo>
        $<$<CONFIG:Release>:-O3>
        $<$<CONFIG:Release>:-lineinfo>
    >
)

# ------------------------------------------------------------------------------
# Static Analysis (Optional)
# ------------------------------------------------------------------------------

if(ONYX_ENABLE_STATIC_ANALYSIS)
    find_program(CLANG_TIDY_EXE NAMES clang-tidy)
    if(CLANG_TIDY_EXE)
        message(STATUS "clang-tidy: ${CLANG_TIDY_EXE}")
        set(CMAKE_CXX_CLANG_TIDY ${CLANG_TIDY_EXE})
    else()
        message(WARNING "clang-tidy not found")
    endif()
endif()

# ------------------------------------------------------------------------------
# Generated Files
# ------------------------------------------------------------------------------

configure_file(
    ${CMAKE_SOURCE_DIR}/cmake/Version.h.in
    ${CMAKE_BINARY_DIR}/generated/Version.h
    @ONLY
)

# ------------------------------------------------------------------------------
# Source File Lists
# ------------------------------------------------------------------------------

set(ONYX_CORE_SOURCES
    src/core/ButterworthFilter.cpp
    src/core/HolonomyCalculator.cpp
    src/core/MotionPath.cpp
    src/core/SpikeFilter.cpp
)

set(ONYX_CUDA_SOURCES
    src/cuda/Kernels.cu
    src/cuda/PhaseCorrelator.cpp
    src/cuda/CudaContext.cpp
)

set(ONYX_OFX_SOURCES
    src/ofx/OnyxPlugin.cpp
    src/ofx/ParameterManager.cpp
)

set(ONYX_UTIL_SOURCES
    src/util/Logger.cpp
)

# ------------------------------------------------------------------------------
# Core Library (Static)
# ------------------------------------------------------------------------------

add_library(onyx_core STATIC
    ${ONYX_CORE_SOURCES}
)

target_include_directories(onyx_core
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/src>
        $<BUILD_INTERFACE:${CMAKE_BINARY_DIR}/generated>
)

target_link_libraries(onyx_core
    PUBLIC
        Threads::Threads
    PRIVATE
        onyx_warnings
)

# ------------------------------------------------------------------------------
# CUDA Library (Static)
# ------------------------------------------------------------------------------

add_library(onyx_cuda STATIC
    ${ONYX_CUDA_SOURCES}
)

target_include_directories(onyx_cuda
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/src>
        $<BUILD_INTERFACE:${CMAKE_BINARY_DIR}/generated>
)

target_link_libraries(onyx_cuda
    PUBLIC
        onyx_core
        CUDA::cudart
        CUDA::cufft
    PRIVATE
        onyx_warnings
        onyx_cuda_options
)

set_target_properties(onyx_cuda PROPERTIES
    CUDA_SEPARABLE_COMPILATION ON
    CUDA_RESOLVE_DEVICE_SYMBOLS ON
)

# ------------------------------------------------------------------------------
# OFX Plugin (Shared Library / Module)
# ------------------------------------------------------------------------------

add_library(OnYX MODULE
    ${ONYX_OFX_SOURCES}
    ${ONYX_UTIL_SOURCES}
)

set_target_properties(OnYX PROPERTIES
    PREFIX ""
    SUFFIX ".ofx"
    OUTPUT_NAME "OnYX"
    # Output directly to bundle structure
    LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/OnYX.ofx.bundle/Contents/Linux-x86-64"
)

target_include_directories(OnYX
    PRIVATE
        ${CMAKE_SOURCE_DIR}/src
        ${CMAKE_BINARY_DIR}/generated
        ${OPENFX_INCLUDE_DIR}
        ${OPENFX_SUPPORT_INCLUDE_DIR}
)

target_link_libraries(OnYX
    PRIVATE
        onyx_core
        onyx_cuda
        onyx_warnings
        CUDA::cudart
        CUDA::cufft
)

target_compile_definitions(OnYX
    PRIVATE
        ONYX_PLUGIN_ID="com.bakerandboyle.onyx"
        ONYX_VERSION_MAJOR=${PROJECT_VERSION_MAJOR}
        ONYX_VERSION_MINOR=${PROJECT_VERSION_MINOR}
        ONYX_VERSION_PATCH=${PROJECT_VERSION_PATCH}
)

# Hide internal symbols, export only OFX entry points
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    target_link_options(OnYX PRIVATE
        -Wl,--version-script=${CMAKE_SOURCE_DIR}/cmake/exports.map
    )
endif()

# ------------------------------------------------------------------------------
# OFX Bundle Creation
# ------------------------------------------------------------------------------

set(BUNDLE_DIR "${CMAKE_BINARY_DIR}/OnYX.ofx.bundle")
set(BUNDLE_CONTENTS "${BUNDLE_DIR}/Contents")

# Create bundle structure
file(MAKE_DIRECTORY "${BUNDLE_CONTENTS}/Resources")
file(MAKE_DIRECTORY "${BUNDLE_CONTENTS}/Linux-x86-64")

# Generate bundle Info.xml
configure_file(
    ${CMAKE_SOURCE_DIR}/cmake/Info.xml.in
    ${BUNDLE_CONTENTS}/Info.xml
    @ONLY
)

# Copy resources
add_custom_command(TARGET OnYX POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        ${CMAKE_SOURCE_DIR}/THIRD_PARTY_LICENSES.txt
        ${BUNDLE_CONTENTS}/Resources/
    COMMENT "Copying bundle resources"
)

# ------------------------------------------------------------------------------
# Tests
# ------------------------------------------------------------------------------

if(ONYX_BUILD_TESTS)
    enable_testing()
    
    include(FetchContent)
    FetchContent_Declare(
        googletest
        GIT_REPOSITORY https://github.com/google/googletest.git
        GIT_TAG v1.14.0
    )
    set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
    set(BUILD_GMOCK OFF CACHE BOOL "" FORCE)
    set(INSTALL_GTEST OFF CACHE BOOL "" FORCE)
    FetchContent_MakeAvailable(googletest)
    
    add_executable(onyx_tests
        tests/core/ButterworthFilterTest.cpp
        tests/core/SpikeFilterTest.cpp
        tests/core/HolonomyCalculatorTest.cpp
        tests/core/MotionPathTest.cpp
        tests/main.cpp
    )
    
    target_include_directories(onyx_tests
        PRIVATE
            ${CMAKE_SOURCE_DIR}/src
            ${CMAKE_BINARY_DIR}/generated
    )
    
    target_link_libraries(onyx_tests
        PRIVATE
            onyx_core
            onyx_cuda
            GTest::gtest
            GTest::gtest_main
    )
    
    include(GoogleTest)
    gtest_discover_tests(onyx_tests)
    
    message(STATUS "Tests: ON")
else()
    message(STATUS "Tests: OFF")
endif()

# ------------------------------------------------------------------------------
# Installation
# ------------------------------------------------------------------------------

# Standard OFX paths
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    set(OFX_PLUGIN_PATH "/usr/OFX/Plugins" CACHE PATH "System OFX plugin directory")
    set(OFX_USER_PATH "$ENV{HOME}/.OFX/Plugins" CACHE PATH "User OFX plugin directory")
elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    set(OFX_PLUGIN_PATH "/Library/OFX/Plugins" CACHE PATH "System OFX plugin directory")
    set(OFX_USER_PATH "$ENV{HOME}/Library/OFX/Plugins" CACHE PATH "User OFX plugin directory")
elseif(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    set(OFX_PLUGIN_PATH "C:/Program Files/Common Files/OFX/Plugins" CACHE PATH "System OFX plugin directory")
    set(OFX_USER_PATH "$ENV{APPDATA}/OFX/Plugins" CACHE PATH "User OFX plugin directory")
endif()

# System-wide install (requires sudo on Linux)
install(
    DIRECTORY ${BUNDLE_DIR}
    DESTINATION ${OFX_PLUGIN_PATH}
    USE_SOURCE_PERMISSIONS
)

# User-local install target (no sudo required)
add_custom_target(install-user
    COMMAND ${CMAKE_COMMAND} -E make_directory "${OFX_USER_PATH}"
    COMMAND ${CMAKE_COMMAND} -E copy_directory
        "${BUNDLE_DIR}"
        "${OFX_USER_PATH}/OnYX.ofx.bundle"
    DEPENDS OnYX
    COMMENT "Installing OnYX to ${OFX_USER_PATH}/OnYX.ofx.bundle"
)

# ------------------------------------------------------------------------------
# Summary
# ------------------------------------------------------------------------------

message(STATUS "")
message(STATUS "========================================")
message(STATUS "OnYX Build Configuration")
message(STATUS "========================================")
message(STATUS "  Version:        ${PROJECT_VERSION}")
message(STATUS "  Git:            ${ONYX_GIT_DESCRIBE}")
message(STATUS "  Build Type:     ${CMAKE_BUILD_TYPE}")
message(STATUS "")
message(STATUS "  C++ Compiler:   ${CMAKE_CXX_COMPILER_ID} ${CMAKE_CXX_COMPILER_VERSION}")
message(STATUS "  CUDA Compiler:  ${CMAKE_CUDA_COMPILER_VERSION}")
message(STATUS "  CUDA Archs:     ${CMAKE_CUDA_ARCHITECTURES}")
message(STATUS "")
message(STATUS "  OpenFX SDK:     ${OPENFX_INCLUDE_DIR}")
message(STATUS "  Bundle Output:  ${BUNDLE_DIR}")
message(STATUS "")
message(STATUS "Targets:")
message(STATUS "  cmake --build .           Build plugin")
message(STATUS "  cmake --build . -t test   Run tests")
message(STATUS "  cmake --build . -t install-user")
message(STATUS "                            Install to ~/.OFX/Plugins")
message(STATUS "  sudo cmake --install .    Install to /usr/OFX/Plugins")
message(STATUS "========================================")
message(STATUS "")

# USAGE
# Include this project transitively in your cmake project using add_subdirectory
# Alternatively, copy the contents of the public folder somewhere in your include
# path and be sure to compile with the correct architecture flags for your compiler
# Search for "target_compile_options" in this file to see what flags are needed.

cmake_minimum_required(VERSION 3.15)

project(klein LANGUAGES C CXX)

# Detect if we are a standalone projection or if we're included transitively
# and set the various option defaults accordingly. By default, tests and utilities
# are not built unless Klein is built as a standalone project
if(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
    set(KLEIN_STANDALONE ON)
    set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
    option(KLEIN_ENABLE_PERF "Enable downloading external libs for perf analysis" ON)
    option(KLEIN_ENABLE_TESTS "Enable compilation of Klein tests" ON)
    option(KLEIN_VALIDATE "Enable runtime validations" ON)
else()
    set(KLEIN_STANDALONE OFF)
    option(KLEIN_ENABLE_PERF "Enable downloading external libs for perf analysis" OFF)
    option(KLEIN_ENABLE_TESTS "Enable compilation of Klein tests" OFF)
    option(KLEIN_VALIDATE "Enable runtime validations" OFF)
endif()

option(KLEIN_BUILD_SYM "Enable compilation of symbolic Klein utility" ON)
option(KLEIN_BUILD_C_BINDINGS "Enable compilation of the Klein C bindings" ON)

# The default platform and instruction set is x86 SSE3
add_library(klein INTERFACE)
add_library(klein::klein ALIAS klein)
target_include_directories(klein INTERFACE public)
if(NOT MSVC)
    target_compile_options(klein INTERFACE -msse3)
endif()

add_library(klein_cxx11 INTERFACE)
add_library(klein::klein_cxx11 ALIAS klein_cxx11)
target_include_directories(klein_cxx11 INTERFACE public)
target_compile_features(klein_cxx11 INTERFACE cxx_std_11)
if(NOT MSVC)
    target_compile_options(klein_cxx11 INTERFACE -msse3)
endif()

add_library(klein_sse42 INTERFACE)
add_library(klein::klein_sse42 ALIAS klein_sse42)
target_include_directories(klein_sse42 INTERFACE public)
target_compile_features(klein_sse42 INTERFACE cxx_std_17)
# SSE4.1 has > 97% market penetration according to the Steam hardware survey
# queried as of December 2019 while AVX2 is around 70%. Thus, we can assume
# FMA support is at least 70%, but perhaps not much more beyond that.
# TODO: Optionally support FMA
if(MSVC)
    # On MSVC, SSE2 enables code generation of SSE2 and later (does not include
    # AVX extensions). This is on by default.
else()
    # Unlike MSVC, FMA instructions are enabled with a separate feature flag
    target_compile_options(klein_sse42 INTERFACE -msse4.1)
    target_compile_definitions(klein_sse42 INTERFACE KLEIN_SSE_4_1)
endif()

if(KLEIN_ENABLE_PERF)
    add_subdirectory(perf)
endif()

if(KLEIN_ENABLE_TESTS)
    enable_testing()

    add_subdirectory(test)
endif()

if(KLEIN_BUILD_SYM)
    add_subdirectory(sym)
endif()

if(KLEIN_BUILD_C_BINDINGS)
    add_subdirectory(c_src)
endif()
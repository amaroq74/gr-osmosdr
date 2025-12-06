########################################################################
# Project setup
########################################################################
# Updated to 3.20 to support modern target-based dependency management
cmake_minimum_required(VERSION 3.20)
project(gr-osmosdr CXX C)

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)
enable_testing()

# Set default build type to Release
if(NOT CMAKE_BUILD_TYPE)
   set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel." FORCE)
   message(STATUS "Build type not specified: defaulting to Release.")
endif()

########################################################################
# GNURadio setup
########################################################################
list(INSERT CMAKE_MODULE_PATH 0 ${CMAKE_SOURCE_DIR}/cmake/Modules)

# Modern GNU Radio (3.10+) prefers finding components via config mode
# Adjust "3.10" to match your specific target version if needed
find_package(Gnuradio "3.10" REQUIRED COMPONENTS blocks fft filter)

# Version Information
set(VERSION_MAJOR 0)
set(VERSION_API   2)
set(VERSION_ABI   0)
set(VERSION_PATCH 0)
include(GrVersion)

########################################################################
# Compiler specific setup
########################################################################

# Modern CMake handles C++ standards via properties rather than definitions
set(CMAKE_CXX_STANDARD 17) # Updated from 11 to 17 for modern GR compatibility
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Visibility settings
set(CMAKE_CXX_VISIBILITY_PRESET hidden)
set(CMAKE_VISIBILITY_INLINES_HIDDEN ON)

# Compiler Warnings
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU")
    add_compile_options(-Wall -Wextra -Wno-unused-parameter -Wsign-compare)
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    add_compile_definitions(NOMINMAX)
    add_compile_definitions(BOOST_ALL_NO_LIB)
endif()

# SIMD Configuration
if(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64|AMD64|x86")
    set(USE_SIMD "SSE2" CACHE STRING "Use SIMD instructions")
else()
    set(USE_SIMD "no" CACHE STRING "Use SIMD instructions")
endif()

set(USE_SIMD_VALUES "no" "SSE2" "AVX")
set_property(CACHE USE_SIMD PROPERTY STRINGS ${USE_SIMD_VALUES})

if(NOT USE_SIMD IN_LIST USE_SIMD_VALUES)
    message(FATAL_ERROR "Option ${USE_SIMD} not supported, valid entries are ${USE_SIMD_VALUES}")
endif()

if(USE_SIMD STREQUAL "SSE2")
    add_compile_definitions(USE_SSE2)
    if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
        add_compile_options(-msse2)
    elseif(MSVC)
        add_compile_options(/arch:SSE2)
    endif()
elseif(USE_SIMD STREQUAL "AVX")
    add_compile_definitions(USE_AVX)
    if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
        add_compile_options(-march=native)
    elseif(MSVC)
        add_compile_options(/arch:AVX)
    endif()
endif()

########################################################################
# Find Dependencies
########################################################################

# Boost
find_package(Boost "1.71" REQUIRED COMPONENTS chrono thread)

include(GrComponent)
set(ENABLE_NONFREE FALSE CACHE BOOL "Enable or disable nonfree components.")

# Hardware Drivers
find_package(LibRTLSDR)
find_package(LibMiriSDR)
if(ENABLE_NONFREE)
    find_package(LibSDRplay)
endif()
find_package(LibHackRF)
find_package(LibAIRSPY)
find_package(LibAIRSPYHF)
find_package(LibbladeRF)
find_package(GnuradioFuncube)
find_package(SoapySDR NO_MODULE)
find_package(LibFreeSRP)
find_package(LibXTRX)

# Python Support (Modernized)
find_package(Python "3.8" COMPONENTS Interpreter Development)
find_package(pybind11)

if(Python_FOUND AND pybind11_FOUND)
    set(ENABLE_PYTHON ON)
else()
    set(ENABLE_PYTHON OFF)
endif()

########################################################################
# Install directories
########################################################################
include(GrPlatform)

if(NOT CMAKE_MODULES_DIR)
    set(CMAKE_MODULES_DIR ${CMAKE_INSTALL_LIBDIR}/cmake)
endif()

set(GR_INCLUDE_DIR      include)
set(GR_CMAKE_DIR        ${CMAKE_MODULES_DIR}/osmosdr)
set(GR_PKG_DATA_DIR     ${GR_DATA_DIR}/${CMAKE_PROJECT_NAME})
set(GR_PKG_DOC_DIR      ${GR_DOC_DIR}/${CMAKE_PROJECT_NAME})
set(GR_PKG_CONF_DIR     ${GR_CONF_DIR}/${CMAKE_PROJECT_NAME}/conf.d)
set(GR_PKG_LIBEXEC_DIR  ${GR_LIBEXEC_DIR}/${CMAKE_PROJECT_NAME})

########################################################################
# Apple RPATH Setup
########################################################################
if(APPLE)
    set(CMAKE_MACOSX_RPATH ON)
    set(CMAKE_SKIP_BUILD_RPATH FALSE)
    set(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)
    set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}")
    set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
endif()

########################################################################
# Doxygen
########################################################################
find_package(Doxygen)
option(ENABLE_DOXYGEN "Build docs using Doxygen" ${DOXYGEN_FOUND})

########################################################################
# Uninstall Target
########################################################################
# Modern way uses standard script usually, but keeping custom for legacy compat
configure_file(
    ${CMAKE_SOURCE_DIR}/cmake/cmake_uninstall.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake
    @ONLY)

add_custom_target(uninstall
    ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake
)

########################################################################
# Subdirectories
########################################################################
add_subdirectory(include/osmosdr)
add_subdirectory(lib)
if(ENABLE_PYTHON)
    add_subdirectory(python)
    add_subdirectory(grc)
    add_subdirectory(apps)
endif()
add_subdirectory(docs)

########################################################################
# Summary
########################################################################
# Note: GR_PRINT_COMPONENT_SUMMARY is a macro provided by GrComponent
# If it is not defined in your GrComponent.cmake, you may need to define it or remove this.
if(COMMAND GR_PRINT_COMPONENT_SUMMARY)
    GR_PRINT_COMPONENT_SUMMARY()
endif()

if(ENABLE_NONFREE)
    message(STATUS "NONFREE components enabled. Binaries cannot be distributed under GPL terms.")
endif()

message(STATUS "Building for version: ${VERSION} / ${LIBVER}")
message(STATUS "Using install prefix: ${CMAKE_INSTALL_PREFIX}")

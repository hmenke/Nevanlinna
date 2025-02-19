# ##############################################################################
#
# Nevanlinna - An example application using triqs and cpp2py
#
# Copyright (C) ...
#
# Nevanlinna is free software: you can redistribute it and/or modify it under the
# terms of the GNU General Public License as published by the Free Software
# Foundation, either version 3 of the License, or (at your option) any later
# version.
#
# Nevanlinna is distributed in the hope that it will be useful, but WITHOUT ANY
# WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR
# A PARTICULAR PURPOSE. See the GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License along with
# Nevanlinna (in the file COPYING.txt in this directory). If not, see
# <http://www.gnu.org/licenses/>.
#
# ##############################################################################

cmake_minimum_required(VERSION 3.20 FATAL_ERROR)
cmake_policy(VERSION 3.20)

# ############
# Define Project
project(triqs_Nevanlinna VERSION 3.2.0 LANGUAGES C CXX)
get_directory_property(IS_SUBPROJECT PARENT_DIRECTORY)

set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)


# Get the git hash & print status
execute_process(COMMAND git rev-parse HEAD WORKING_DIRECTORY ${PROJECT_SOURCE_DIR} OUTPUT_VARIABLE PROJECT_GIT_HASH OUTPUT_STRIP_TRAILING_WHITESPACE)
message(STATUS "${PROJECT_NAME} version : ${PROJECT_VERSION}")
message(STATUS "${PROJECT_NAME} Git hash: ${PROJECT_GIT_HASH}")

# ############
# Load TRIQS
option(TRIQS_Support "TRIQS Support" ON)
if(TRIQS_Support)
  find_package(TRIQS 3.1 REQUIRED)
  message(STATUS "Compiling with TRIQS Support")
  
  # Enforce Consistent Versioning
  if(NOT ${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR} VERSION_EQUAL ${TRIQS_VERSION_MAJOR}.${TRIQS_VERSION_MINOR})
    message(FATAL_ERROR "The ${PROJECT_NAME} version ${PROJECT_VERSION} is not compatible with TRIQS version ${TRIQS_VERSION}.")
  endif()
else()
  message(STATUS "Compiling without TRIQS Support! Limited functionality")
endif()

# Default Install directory to TRIQS_ROOT if not given or invalid.
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT OR (NOT IS_ABSOLUTE ${CMAKE_INSTALL_PREFIX}))
  if(TRIQS_Support)
    message(STATUS "No install prefix given (or invalid). Defaulting to TRIQS_ROOT")
    set(CMAKE_INSTALL_PREFIX ${TRIQS_ROOT} CACHE PATH "default install path" FORCE)
    set(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT FALSE)
  else()
  message(FATAL_ERROR "No install prefix given (or invalid)")
  endif()
endif()
if(NOT IS_SUBPROJECT)
  message(STATUS "-------- CMAKE_INSTALL_PREFIX: ${CMAKE_INSTALL_PREFIX} --------")
endif()
set(${PROJECT_NAME}_BINARY_DIR ${PROJECT_BINARY_DIR} CACHE STRING "Binary directory of the ${PROJECT_NAME} Project")

# Make additional Find Modules available
list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/share/cmake/Modules)

# ############
# CMake Options

# Default to Release build type
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Type of build" FORCE)
endif()
if(NOT IS_SUBPROJECT)
  message(STATUS "-------- BUILD-TYPE: ${CMAKE_BUILD_TYPE} --------")
endif()

# Python Support
option(PythonSupport "Build with Python support" ON)

# Documentation
option(Build_Documentation "Build documentation" OFF)
if(Build_Documentation AND NOT PythonSupport)
  message(FATAL_ERROR "Build_Documentation=ON requires PythonSupport to be enabled")
endif()

# Testing
option(Build_Tests "Build tests" ON)
if(Build_Tests)
  enable_testing()
endif()

option(Build_Examples "Build examples" ON)

# Build static libraries by default
option(BUILD_SHARED_LIBS "Enable compilation of shared libraries" OFF)

# ############
# Global Compilation Settings

# Export the list of compile-commands into compile_commands.json
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Disable compiler extensions
set(CMAKE_CXX_EXTENSIONS OFF)

# Provide additional debugging information for Debug builds
add_compile_options($<$<CONFIG:Debug>:-ggdb3>)

# Create an Interface target for compiler warnings
add_library(${PROJECT_NAME}_warnings INTERFACE)
target_compile_options(${PROJECT_NAME}_warnings
  INTERFACE
    -Wall
    -Wextra
    -Wfloat-conversion
    -Wpedantic
    -Wno-sign-compare
    $<$<CXX_COMPILER_ID:GNU>:-Wno-comma-subscript>
    $<$<CXX_COMPILER_ID:GNU>:-Wshadow=local>
    $<$<CXX_COMPILER_ID:GNU>:-Wno-attributes>
    $<$<CXX_COMPILER_ID:GNU>:-Wno-deprecated-declarations>
    $<$<CXX_COMPILER_ID:AppleClang,Clang,IntelLLVM>:-Wno-deprecated-comma-subscript>
    $<$<CXX_COMPILER_ID:AppleClang,Clang,IntelLLVM>:-Wno-unknown-warning-option>
    $<$<CXX_COMPILER_ID:AppleClang,Clang,IntelLLVM>:-Wshadow>
    $<$<CXX_COMPILER_ID:AppleClang,Clang,IntelLLVM>:-Wno-gcc-compat>
    $<$<CXX_COMPILER_ID:AppleClang,Clang,IntelLLVM>:-Wno-c++20-extensions>
    $<$<CXX_COMPILER_ID:AppleClang,Clang,IntelLLVM>:-Wno-c++20-compat>
    $<$<CXX_COMPILER_ID:IntelLLVM>:-Wno-tautological-constant-compare>
)

# #############
# Build Project

# Find / Build dependencies
add_subdirectory(deps)

# Build and install the library
add_subdirectory(c++/${PROJECT_NAME})

# Tests
if(Build_Tests)
  add_subdirectory(test)
endif()

# Python
if(PythonSupport)
  add_subdirectory(python/${PROJECT_NAME})
endif()

# Docs
if(NOT IS_SUBPROJECT AND Build_Documentation)
  add_subdirectory(doc)
endif()

if(Build_Examples)
  add_subdirectory(examples)
endif()

# Additional configuration files
add_subdirectory(share)

# #############
# Debian Package

option(BUILD_DEBIAN_PACKAGE "Build a deb package" OFF)
if(BUILD_DEBIAN_PACKAGE AND NOT IS_SUBPROJECT)
  if(NOT CMAKE_INSTALL_PREFIX STREQUAL "/usr")
    message(FATAL_ERROR "CMAKE_INSTALL_PREFIX must be /usr for packaging")
  endif()
  set(CPACK_PACKAGE_NAME ${PROJECT_NAME})
  set(CPACK_GENERATOR "DEB")
  set(CPACK_PACKAGE_VERSION ${PROJECT_VERSION})
  set(CPACK_PACKAGE_CONTACT "https://github.com/TRIQS/Nevanlinna")
  execute_process(COMMAND dpkg --print-architecture OUTPUT_VARIABLE CMAKE_DEBIAN_PACKAGE_ARCHITECTURE OUTPUT_STRIP_TRAILING_WHITESPACE)
  set(CPACK_DEBIAN_PACKAGE_DEPENDS "triqs (>= 3.2)")
  set(CPACK_DEBIAN_PACKAGE_SHLIBDEPS ON)
  set(CPACK_DEBIAN_PACKAGE_GENERATE_SHLIBS ON)
  include(CPack)
endif()

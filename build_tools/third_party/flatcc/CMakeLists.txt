# Copyright 2020 The IREE Authors
#
# Licensed under the Apache License v2.0 with LLVM Exceptions.
# See https://llvm.org/LICENSE.txt for license information.
# SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception

set(FLATCC_ROOT "${IREE_ROOT_DIR}/third_party/flatcc/")

# We don't install any headers because their only use is in built code
# (not public).
set(IREE_HDRS_ROOT_PATH OFF)
# Considered part of the runtime. See runtime/src/CMakeLists.txt.
set(IREE_INSTALL_LIBRARY_TARGETS_DEFAULT_COMPONENT IREEBundledLibraries)
set(IREE_INSTALL_LIBRARY_TARGETS_DEFAULT_EXPORT_SET Runtime)

external_cc_library(
  PACKAGE
    flatcc
  NAME
    runtime
  ROOT
    ${FLATCC_ROOT}
  INCLUDES
    "${FLATCC_ROOT}/include"
  SRCS
    "src/runtime/builder.c"
    "src/runtime/emitter.c"
    "src/runtime/json_parser.c"
    "src/runtime/json_printer.c"
    "src/runtime/refmap.c"
  HDRS
    "include/flatcc/flatcc_accessors.h"
    "include/flatcc/flatcc_alloc.h"
    "include/flatcc/flatcc_assert.h"
    "include/flatcc/flatcc_builder.h"
    "include/flatcc/flatcc_emitter.h"
    "include/flatcc/flatcc_endian.h"
    "include/flatcc/flatcc_epilogue.h"
    "include/flatcc/flatcc_flatbuffers.h"
    "include/flatcc/flatcc_identifier.h"
    "include/flatcc/flatcc_iov.h"
    "include/flatcc/flatcc_json_parser.h"
    "include/flatcc/flatcc_json_printer.h"
    "include/flatcc/flatcc_portable.h"
    "include/flatcc/flatcc_prologue.h"
    "include/flatcc/flatcc_refmap.h"
    "include/flatcc/flatcc_rtconfig.h"
    "include/flatcc/flatcc_types.h"
    "include/flatcc/flatcc_unaligned.h"
    "include/flatcc/flatcc_verifier.h"
    "include/flatcc/reflection/flatbuffers_common_builder.h"
    "include/flatcc/reflection/flatbuffers_common_reader.h"
  DEPS
    flatcc::parsing
  PUBLIC
)

# A limited version of :runtime with only enough to parse and verify.
external_cc_library(
  PACKAGE
    flatcc
  NAME
    parsing
  ROOT
    ${FLATCC_ROOT}
  SYSTEM_INCLUDES
    "${FLATCC_ROOT}/include"
  SRCS
    "config/config.h"
    "src/runtime/verifier.c"
  HDRS
    "include/flatcc/flatcc_accessors.h"
    "include/flatcc/flatcc_alloc.h"
    "include/flatcc/flatcc_assert.h"
    "include/flatcc/flatcc_endian.h"
    "include/flatcc/flatcc_epilogue.h"
    "include/flatcc/flatcc_flatbuffers.h"
    "include/flatcc/flatcc_identifier.h"
    "include/flatcc/flatcc_portable.h"
    "include/flatcc/flatcc_prologue.h"
    "include/flatcc/flatcc_rtconfig.h"
    "include/flatcc/flatcc_types.h"
    "include/flatcc/flatcc_unaligned.h"
    "include/flatcc/flatcc_verifier.h"
    "include/flatcc/reflection/flatbuffers_common_reader.h"
  PUBLIC
)

# The iree-flatcc-cli tool runs on the build host to generate flatbuffer
# schema headers at build time. When cross-compiling without prebuilt host
# tools (IREE_HOST_BIN_DIR) a cross-compiled binary cannot execute on the
# host (and freestanding targets such as wasm32 cannot even compile it), so
# bootstrap a host copy at configure time using the default host toolchain.
if(CMAKE_CROSSCOMPILING AND NOT IREE_HOST_BIN_DIR)
  set(_FLATCC_HOST_BUILD_DIR "${IREE_BINARY_DIR}/host_tools/flatcc")
  set(_HOST_EXECUTABLE_SUFFIX "")
  if(CMAKE_HOST_SYSTEM_NAME STREQUAL "Windows")
    set(_HOST_EXECUTABLE_SUFFIX ".exe")
  endif()
  set(_FLATCC_HOST_TOOL
      "${_FLATCC_HOST_BUILD_DIR}/iree-flatcc-cli${_HOST_EXECUTABLE_SUFFIX}")
  if(NOT EXISTS "${_FLATCC_HOST_TOOL}")
    message(STATUS "Building host iree-flatcc-cli for cross-compiled flatbuffer schema generation")
    execute_process(
      COMMAND
        "${CMAKE_COMMAND}"
        -S "${CMAKE_CURRENT_SOURCE_DIR}/host"
        -B "${_FLATCC_HOST_BUILD_DIR}"
        "-DFLATCC_ROOT=${FLATCC_ROOT}"
        -DCMAKE_BUILD_TYPE=Release
      RESULT_VARIABLE _FLATCC_HOST_CONFIGURE_RESULT
    )
    if(NOT _FLATCC_HOST_CONFIGURE_RESULT EQUAL 0)
      message(FATAL_ERROR "Configuring host iree-flatcc-cli failed (${_FLATCC_HOST_CONFIGURE_RESULT})")
    endif()
    execute_process(
      COMMAND "${CMAKE_COMMAND}" --build "${_FLATCC_HOST_BUILD_DIR}" --config Release
      RESULT_VARIABLE _FLATCC_HOST_BUILD_RESULT
    )
    if(NOT _FLATCC_HOST_BUILD_RESULT EQUAL 0)
      message(FATAL_ERROR "Building host iree-flatcc-cli failed (${_FLATCC_HOST_BUILD_RESULT})")
    endif()
  endif()
  set(IREE_FLATCC_HOST_TOOL "${_FLATCC_HOST_TOOL}" CACHE FILEPATH
      "Host flatcc compiler used for schema generation when cross-compiling." FORCE)
  # The cross-compiled CLI below is not built: it is useless on the target and
  # freestanding toolchains cannot compile it.
  return()
endif()

# The iree-flatcc-cli tool depends on POSIX APIs (e.g., posix_memalign).
# Skip building it when POSIX support is disabled and import the host binary instead.
if(NOT IREE_ENABLE_POSIX)
  if(IREE_HOST_BIN_DIR)
    iree_import_binary(NAME iree-flatcc-cli)
  endif()
  return()
endif()

# Define our own binary target for the CLI.
# flatcc's `flatcc_cli` target renames itself to `flatcc` or `flatcc_d`
# depending on the build configuration, so we prefer to just set our own name.
add_executable(iree-flatcc-cli
  "${FLATCC_ROOT}/src/cli/flatcc_cli.c"
  "${FLATCC_ROOT}/external/hash/cmetrohash64.c"
  "${FLATCC_ROOT}/external/hash/str_set.c"
  "${FLATCC_ROOT}/external/hash/ptr_set.c"
  "${FLATCC_ROOT}/src/compiler/hash_tables/symbol_table.c"
  "${FLATCC_ROOT}/src/compiler/hash_tables/scope_table.c"
  "${FLATCC_ROOT}/src/compiler/hash_tables/name_table.c"
  "${FLATCC_ROOT}/src/compiler/hash_tables/schema_table.c"
  "${FLATCC_ROOT}/src/compiler/hash_tables/value_set.c"
  "${FLATCC_ROOT}/src/compiler/fileio.c"
  "${FLATCC_ROOT}/src/compiler/parser.c"
  "${FLATCC_ROOT}/src/compiler/semantics.c"
  "${FLATCC_ROOT}/src/compiler/coerce.c"
  "${FLATCC_ROOT}/src/compiler/codegen_schema.c"
  "${FLATCC_ROOT}/src/compiler/flatcc.c"
  "${FLATCC_ROOT}/src/compiler/codegen_c.c"
  "${FLATCC_ROOT}/src/compiler/codegen_c_reader.c"
  "${FLATCC_ROOT}/src/compiler/codegen_c_sort.c"
  "${FLATCC_ROOT}/src/compiler/codegen_c_builder.c"
  "${FLATCC_ROOT}/src/compiler/codegen_c_verifier.c"
  "${FLATCC_ROOT}/src/compiler/codegen_c_sorter.c"
  "${FLATCC_ROOT}/src/compiler/codegen_c_json_parser.c"
  "${FLATCC_ROOT}/src/compiler/codegen_c_json_printer.c"
  "${FLATCC_ROOT}/src/runtime/builder.c"
  "${FLATCC_ROOT}/src/runtime/emitter.c"
  "${FLATCC_ROOT}/src/runtime/refmap.c"
)
set_target_properties(iree-flatcc-cli
  PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY "${IREE_BINARY_DIR}/tools"
)
target_include_directories(iree-flatcc-cli SYSTEM
  PUBLIC
    "${FLATCC_ROOT}/external"
    "${FLATCC_ROOT}/include"
    "${FLATCC_ROOT}/config"
)
install(
  TARGETS iree-flatcc-cli
  COMPONENT IREETools-Runtime
  RUNTIME DESTINATION bin
  BUNDLE DESTINATION bin
)

# Copyright 2026 The IREE Authors
#
# Licensed under the Apache License v2.0 with LLVM Exceptions.
# See https://llvm.org/LICENSE.txt for license information.
# SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception

# Standalone host build of the flatcc compiler CLI.
#
# This is used to bootstrap flatbuffer schema generation when cross-compiling
# without IREE_HOST_BIN_DIR: schema codegen runs on the build host where a
# cross-compiled binary cannot execute (and freestanding targets such as
# wasm32 cannot even compile it). It is configured and built at configure time
# by build_tools/third_party/flatcc/CMakeLists.txt using the default host
# toolchain.
#
# The source list must be kept in sync with the iree-flatcc-cli target in
# build_tools/third_party/flatcc/CMakeLists.txt.

cmake_minimum_required(VERSION 3.21...3.24)
project(iree_flatcc_host C)

if(NOT FLATCC_ROOT)
  get_filename_component(FLATCC_ROOT
    "${CMAKE_CURRENT_SOURCE_DIR}/../../../../third_party/flatcc" ABSOLUTE)
endif()

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
target_include_directories(iree-flatcc-cli SYSTEM
  PRIVATE
    "${FLATCC_ROOT}/external"
    "${FLATCC_ROOT}/include"
    "${FLATCC_ROOT}/config"
)
# "$<1:...>" suppresses the per-config subdirectory that multi-config
# generators would otherwise append, giving a predictable output path.
set_target_properties(iree-flatcc-cli
  PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY "$<1:${CMAKE_BINARY_DIR}>"
)

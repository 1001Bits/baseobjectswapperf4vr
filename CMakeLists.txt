cmake_minimum_required(VERSION 3.23)

set(NAME "BaseObjectSwapperVR")
set(VERSION 2.2.0)

message(">>> Start build for '${NAME}' v${VERSION}")

set(ROOT_DIR "${CMAKE_CURRENT_SOURCE_DIR}")

if(PROJECT_SOURCE_DIR STREQUAL PROJECT_BINARY_DIR)
  message(FATAL_ERROR ">> In-source builds not allowed.")
endif()

# ── Options ────────────────────────────────────────────────────────────────
option(COPY_BUILD "Copy the build output to the Fallout 4 VR directory." TRUE)

# ── vcpkg setup ────────────────────────────────────────────────────────────
macro(set_from_environment VARIABLE)
  if(NOT DEFINED ${VARIABLE} AND DEFINED ENV{${VARIABLE}})
    set(${VARIABLE} $ENV{${VARIABLE}})
  endif()
endmacro()

set_from_environment(VCPKG_ROOT)
set_from_environment(Fallout4VRPath)
set_from_environment(CommonLibF4Path)

# Pull our local clib-util port overlay (matches the upstream NG repo layout).
set(VCPKG_OVERLAY_PORTS "${CMAKE_CURRENT_SOURCE_DIR}/cmake/ports" CACHE STRING "")

if(DEFINED VCPKG_ROOT)
  set(CMAKE_TOOLCHAIN_FILE "${VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake" CACHE STRING "")
  set(VCPKG_TARGET_TRIPLET "x64-windows-static" CACHE STRING "")
  set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>" CACHE STRING "")
else()
  message(FATAL_ERROR ">> Variable VCPKG_ROOT is not set!")
endif()

project(${NAME} VERSION ${VERSION} LANGUAGES CXX)
message(">>> Building project '${PROJECT_NAME}' v${PROJECT_VERSION}")

# Generated Version.h + version.rc
configure_file(
  "${ROOT_DIR}/cmake/Version.h.in"
  "${CMAKE_CURRENT_BINARY_DIR}/include/Version.h"
  @ONLY
)
configure_file(
  "${ROOT_DIR}/cmake/version.rc.in"
  "${CMAKE_CURRENT_BINARY_DIR}/version.rc"
  @ONLY
)

add_compile_definitions(
  _CRT_SECURE_NO_WARNINGS
  ENABLE_FALLOUT_VR        # VR address library + CSV loader
)
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ON)
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION_DEBUG OFF)

# ── CommonLibF4 (VR-aware fork) ────────────────────────────────────────────
set(CommonLibPath "${ROOT_DIR}/external/CommonLibF4/CommonLibF4")
if(NOT EXISTS "${CommonLibPath}/include/REL/Relocation.h")
  if(DEFINED CommonLibF4Path AND IS_DIRECTORY "${CommonLibF4Path}")
    set(CommonLibPath "${CommonLibF4Path}")
  else()
    # Fallback: known good local fork that supports VR via ENABLE_FALLOUT_VR.
    set(CommonLibPath "C:/Development/Buffout4/Buffout4-master/external/CommonLibF4/CommonLibF4")
  endif()
endif()

if(IS_DIRECTORY "${CommonLibPath}")
  add_subdirectory("${CommonLibPath}" CommonLibF4)
else()
  message(FATAL_ERROR "CommonLibF4 not found. Set CommonLibF4Path or add as submodule under external/.")
endif()

find_package(spdlog REQUIRED CONFIG)
find_package(xbyak REQUIRED CONFIG)
find_package(unordered_dense CONFIG REQUIRED)
find_path(SRELL_INCLUDE_DIRS "srell.hpp")
find_path(CLIBUTIL_INCLUDE_DIRS "CLibUtil/string.hpp")

# ── Sources ────────────────────────────────────────────────────────────────
include(cmake/sourcelist.cmake)

source_group(
  TREE ${CMAKE_CURRENT_SOURCE_DIR}
  FILES ${SOURCES}
)
source_group(
  TREE ${CMAKE_CURRENT_BINARY_DIR}
  FILES ${CMAKE_CURRENT_BINARY_DIR}/include/Version.h
)

add_library(${PROJECT_NAME} SHARED
  ${SOURCES}
  ${CMAKE_CURRENT_BINARY_DIR}/version.rc
)

target_compile_features(${PROJECT_NAME} PRIVATE cxx_std_23)
target_precompile_headers(${PROJECT_NAME} PRIVATE "${ROOT_DIR}/include/PCH.h")

target_include_directories(${PROJECT_NAME} PRIVATE
  "${CMAKE_CURRENT_BINARY_DIR}/include"
  "${ROOT_DIR}/include"
  ${SRELL_INCLUDE_DIRS}
  ${CLIBUTIL_INCLUDE_DIRS}
)

target_link_libraries(${PROJECT_NAME} PRIVATE
  CommonLibF4::CommonLibF4
  spdlog::spdlog
  xbyak::xbyak
  unordered_dense::unordered_dense
)

target_compile_definitions(${PROJECT_NAME} PRIVATE
  UNICODE _UNICODE NOMINMAX WIN32_LEAN_AND_MEAN
  F4SE_SUPPORT_XBYAK
  F4SEVR_SUPPORT
)

target_compile_definitions(CommonLibF4 PUBLIC F4SE_SUPPORT_XBYAK)

if(MSVC)
  target_compile_options(${PROJECT_NAME} PRIVATE
    /sdl
    /utf-8
    /Zi
    /permissive-
    /Zc:preprocessor
    /W4
    /wd4200
    /wd4201
    /wd4324
    "$<$<CONFIG:RELEASE>:/Zc:inline;/JMC-;/Ob3>"
  )
  target_link_options(${PROJECT_NAME} PRIVATE
    "$<$<CONFIG:DEBUG>:/INCREMENTAL;/OPT:NOREF;/OPT:NOICF>"
    "$<$<CONFIG:RELEASE>:/INCREMENTAL:NO;/OPT:REF;/OPT:ICF;/DEBUG:FULL>"
  )
endif()

# ── Package / copy into game install ───────────────────────────────────────
add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
  COMMAND ${CMAKE_COMMAND} -E make_directory "${ROOT_DIR}/package/Data/F4SE/Plugins"
  COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:${PROJECT_NAME}> "${ROOT_DIR}/package/Data/F4SE/Plugins/"
  COMMENT "Packaging ${PROJECT_NAME}..."
)

if(COPY_BUILD AND DEFINED Fallout4VRPath)
  add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:${PROJECT_NAME}> "${Fallout4VRPath}/Data/F4SE/Plugins/"
    COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_PDB_FILE:${PROJECT_NAME}> "${Fallout4VRPath}/Data/F4SE/Plugins/"
    COMMENT "Deploying to Fallout 4 VR install..."
  )
endif()

message(">>> ${PROJECT_NAME} configured for Fallout 4 VR (1.2.72)")

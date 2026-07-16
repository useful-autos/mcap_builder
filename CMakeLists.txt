# =================================================================================================
#  Copyright (C) 2023-2024 MCAP_BUILDER Contributors
# =================================================================================================

cmake_minimum_required(VERSION 3.22.1)

# VERSION is the upstream mcap C++ release this repo wraps. It MUST name an
# existing releases/cpp/v<VERSION> tag on the useful-autos/mcap fork (that is
# what FetchContent pulls below). It is derived from the git tag so the package
# version and the fetched source can never drift; CI passes -DVERSION for its
# detached-tag checkout, and local builds fall back to the nearest tag.
if(NOT DEFINED VERSION)
  find_package(Git REQUIRED)
  execute_process(
    COMMAND ${GIT_EXECUTABLE} describe --tags --abbrev=0
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    OUTPUT_VARIABLE VERSION
    OUTPUT_STRIP_TRAILING_WHITESPACE
    RESULT_VARIABLE _mcap_git_rc)
  if(NOT _mcap_git_rc EQUAL 0)
    message(FATAL_ERROR "No git tag found; pass -DVERSION=<x.y.z> explicitly")
  endif()
endif()
string(REGEX REPLACE "^v" "" VERSION "${VERSION}")

# Debian packaging revision. The git tag stays pinned to the upstream mcap
# version (it mirrors the fork's releases/cpp/v<VERSION> tag), so a
# packaging-only rebuild FORCE-MOVES THE SAME TAG. Whenever you force-move the
# tag, bump this by 1 so apt sees the rebuild as newer. Committed here -- not
# derived from apt or Jenkins state -- so the version is reproducible and
# survives an apt-database wipe or a Jenkins reset.
set(PACKAGE_REVISION 2)

project(mcap_builder LANGUAGES CXX VERSION ${VERSION})

# Full Debian version (<upstream>-<revision>) for the CI packaging step to read.
file(WRITE "${CMAKE_BINARY_DIR}/deb_version.txt" "${VERSION}-${PACKAGE_REVISION}")

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

include(FetchContent)

if(${CMAKE_VERSION} VERSION_GREATER_EQUAL "3.24")
  set(DOWNLOAD_EXTRACT_TIMESTAMP_ARG "DOWNLOAD_EXTRACT_TIMESTAMP;TRUE")
else()
  set(DOWNLOAD_EXTRACT_TIMESTAMP_ARG "")
endif()
set(MCAP_TAG "releases/cpp/v${VERSION}")
FetchContent_Declare(
  mcap
  URL https://github.com/useful-autos/mcap/archive/refs/tags/${MCAP_TAG}.tar.gz
  ${DOWNLOAD_EXTRACT_TIMESTAMP_ARG}
)
FetchContent_MakeAvailable(mcap)
set(MCAP_INCLUDE_DIR ${mcap_SOURCE_DIR}/cpp/mcap/include/)

add_subdirectory(src)


include(CMakePackageConfigHelpers)
write_basic_package_version_file(
    "${PROJECT_BINARY_DIR}/mcap-config-version.cmake"
    VERSION ${CMAKE_PROJECT_VERSION}
    COMPATIBILITY AnyNewerVersion
)

install(TARGETS mcap
    EXPORT mcap-targets
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
    INCLUDES
    DESTINATION include
)
install(DIRECTORY ${MCAP_INCLUDE_DIR} DESTINATION include)

include(CMakePackageConfigHelpers)
configure_package_config_file(
    "${PROJECT_SOURCE_DIR}/cmake/mcap-config.cmake.in"
    "${PROJECT_BINARY_DIR}/mcap-config.cmake"
    INSTALL_DESTINATION lib/cmake/mcap
)

install(EXPORT mcap-targets DESTINATION lib/cmake/mcap)
install(FILES "${PROJECT_BINARY_DIR}/mcap-config-version.cmake"
              "${PROJECT_BINARY_DIR}/mcap-config.cmake"
        DESTINATION lib/cmake/mcap)

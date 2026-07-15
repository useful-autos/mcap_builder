add_library(mcap lib.cpp)

target_include_directories(mcap PUBLIC
  $<BUILD_INTERFACE:${MCAP_INCLUDE_DIR}>
  $<INSTALL_INTERFACE:include/>
)

set(DEPENDENCIES "")

find_package(PkgConfig REQUIRED)

pkg_check_modules(LZ4 REQUIRED IMPORTED_TARGET liblz4)
if (LZ4_FOUND)
  message(STATUS "Found lz4 ${LZ4_VERSION}")
  list(APPEND DEPENDENCIES PkgConfig::LZ4)
else()
  message(WARNING "Dependency LZ4 not found, disabling")
  target_compile_definitions(mcap PUBLIC "MCAP_COMPRESSION_NO_LZ4")
endif()

pkg_check_modules(ZSTD REQUIRED IMPORTED_TARGET libzstd)
if (ZSTD_FOUND)
  message(STATUS "Found zstd ${ZSTD_VERSION}")
  list(APPEND DEPENDENCIES PkgConfig::ZSTD)
else()
  message(WARNING "Dependency zstd not found, disabling")
  target_compile_definitions(mcap PUBLIC "MCAP_COMPRESSION_NO_ZSTD")
endif()

target_link_libraries(mcap PUBLIC ${DEPENDENCIES})

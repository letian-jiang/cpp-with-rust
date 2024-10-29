cmake_minimum_required(VERSION 3.22)
project(cpp_with_rust)

if(CMAKE_BUILD_TYPE STREQUAL "")
    set(CMAKE_BUILD_TYPE "Release")
endif()

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CMAKE_CXX_STANDARD 17)

set(CARGO_MANIFEST ${CMAKE_SOURCE_DIR}/Cargo.toml)
set(CARGO_TARGET_DIR ${CMAKE_SOURCE_DIR}/target)

set(BLOBSTORE_SOURCE_FILE ${CMAKE_SOURCE_DIR}/src/lib.rs)
set(BLOBSTORE_BRIDGE_CPP ${CARGO_TARGET_DIR}/cxxbridge/blobstore/src/lib.rs.cc)

if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(CARGO_BUILD_MODE "--debug")
    set(BLOBSTORE_LIB ${CARGO_TARGET_DIR}/debug/${CMAKE_STATIC_LIBRARY_PREFIX}blobstore${CMAKE_STATIC_LIBRARY_SUFFIX})
else()
    set(CARGO_BUILD_MODE "--release")
    set(BLOBSTORE_LIB ${CARGO_TARGET_DIR}/release/${CMAKE_STATIC_LIBRARY_PREFIX}blobstore${CMAKE_STATIC_LIBRARY_SUFFIX})
endif()

message("CMAKE_BUILD_TYPE: ${CMAKE_BUILD_TYPE}")
message("CMAKE_SOURCE_DIR: ${CMAKE_SOURCE_DIR}")
message("CMAKE_BINARY_DIR: ${CMAKE_BINARY_DIR}")
message("CARGO_BUILD_MODE: ${CARGO_BUILD_MODE}")
message("BLOBSTORE_BRIDGE_CPP: ${BLOBSTORE_BRIDGE_CPP}")
message("BLOBSTORE_LIB: ${BLOBSTORE_LIB}")

# Add a custom command that builds the rust crate and generates C++ bridge code
add_custom_command(
        OUTPUT ${BLOBSTORE_BRIDGE_CPP} ${BLOBSTORE_LIB}
        COMMAND cargo build ${CARGO_BUILD_MODE} --manifest-path ${CARGO_MANIFEST}
        DEPENDS ${BLOBSTORE_SOURCE_FILE}
        USES_TERMINAL
        COMMENT "Running cargo..."
)

add_executable(cpp_with_rust src/main.cpp src/multibuf.cpp ${BLOBSTORE_BRIDGE_CPP})

target_include_directories(
        cpp_with_rust
        PRIVATE
        include/
        ${CARGO_TARGET_DIR}/cxxbridge/blobstore/src/
)

target_link_libraries(cpp_with_rust ${BLOBSTORE_LIB})

set_target_properties(
        cpp_with_rust
        PROPERTIES ADDITIONAL_CLEAN_FILES ${CARGO_TARGET_DIR}
)

# Windows-only configuration
if(WIN32)
    target_link_libraries(cpp_with_rust userenv ws2_32 bcrypt)
    set_target_properties(
            cpp_with_rust
            PROPERTIES
            MSVC_RUNTIME_LIBRARY "MultiThreadedDLL"
            RUNTIME_OUTPUT_DIRECTORY_DEBUG ${CMAKE_BINARY_DIR}
            RUNTIME_OUTPUT_DIRECTORY_RELEASE ${CMAKE_BINARY_DIR}
    )
endif()

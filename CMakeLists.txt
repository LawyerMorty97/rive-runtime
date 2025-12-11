cmake_minimum_required(VERSION 3.20)
project(rive-runtime CXX)

find_program(PREMAKE_EXECUTABLE premake5 REQUIRED)
if(WIN32)
    find_program(MSBUILD_EXECUTABLE msbuild REQUIRED)
endif()

# Map to premake config names
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(RIVE_CFG debug)
else()
    set(RIVE_CFG release)
endif()

set(RIVE_ROOT ${CMAKE_CURRENT_SOURCE_DIR})

set(RIVE_BUILD_DIR ${RIVE_ROOT}/build)
set(RIVE_RENDERER_DIR ${RIVE_ROOT}/renderer)

set(RIVE_INCLUDE_DIR ${RIVE_ROOT}/include)
set(RIVE_DECODERS_INCLUDE_DIR ${RIVE_ROOT}/decoders/include)
set(RIVE_RENDERER_INCLUDE_DIR ${RIVE_ROOT}/renderer/include)

# This is where the rive-runtime libraries are compiled to
set(RIVE_LIB_DIR ${RIVE_RENDERER_DIR}/out/${RIVE_CFG})
if(WIN32)
    set(RIVE_RENDERER_LIB ${RIVE_LIB_DIR}/rive.lib)
    set(RIVE_DECODERS_LIB ${RIVE_LIB_DIR}/rive_decoders.lib)
    set(RIVE_PLS_RENDERER_LIB ${RIVE_LIB_DIR}/rive_pls_renderer.lib)
    set(RIVE_HARFBUZZ_LIB ${RIVE_LIB_DIR}/rive_harfbuzz.lib)
    set(RIVE_SHEENBIDI_LIB ${RIVE_LIB_DIR}/rive_sheenbidi.lib)
    set(RIVE_YOGA_LIB ${RIVE_LIB_DIR}/rive_yoga.lib)

    set(RIVE_BUILD_SCRIPT build_rive.bat)
    set(RIVE_BUILD_FLAGS --windows_runtime=static)
elseif(APPLE)
    set(RIVE_RENDERER_LIB ${RIVE_LIB_DIR}/librive.a)
    set(RIVE_DECODERS_LIB ${RIVE_LIB_DIR}/librive_decoders.a)
    set(RIVE_PLS_RENDERER_LIB ${RIVE_LIB_DIR}/librive_pls_renderer.a)
    set(RIVE_HARFBUZZ_LIB ${RIVE_LIB_DIR}/librive_harfbuzz.a)
    set(RIVE_SHEENBIDI_LIB ${RIVE_LIB_DIR}/librive_sheenbidi.a)
    set(RIVE_YOGA_LIB ${RIVE_LIB_DIR}/librive_yoga.a)
    set(RIVE_BUILD_SCRIPT build_rive.sh)
    set(RIVE_BUILD_FLAGS --with-rtti)
endif()

# Step 0 (windows only): Run `/build/setup_windows_dev.bat`
if(WIN32)
    add_custom_target(rive_prebuild ALL
        COMMAND ${CMAKE_COMMAND} -E echo "[Rive] Preconfiguring rive-runtime..."
        COMMAND setup_windows_dev.bat
        COMMAND powershell -ExecutionPolicy Bypass -File "${RIVE_BUILD_DIR}/setup_windows_dev.ps1"
        WORKING_DIRECTORY ${RIVE_BUILD_DIR}
    )
else()
    add_custom_target(rive_prebuild ALL
        COMMAND ${CMAKE_COMMAND} -E echo "[Rive] Skipping preconfiguring of rive-runtime..."
        WORKING_DIRECTORY ${RIVE_BUILD_DIR}
    )
endif()

# Step 1: Build rive-runtime
add_custom_command(
    OUTPUT
        ${RIVE_RENDERER_LIB}
        ${RIVE_DECODERS_LIB}
        ${RIVE_PLS_RENDERER_LIB}
        ${RIVE_HARFBUZZ_LIB}
        ${RIVE_SHEENBIDI_LIB}
        ${RIVE_YOGA_LIB}
    COMMAND ${CMAKE_COMMAND} -E echo "[Rive] Building rive-runtime - config: ${RIVE_CFG}"
    # COMMAND cmd /c ".\\${RIVE_BUILD_SCRIPT} ${RIVE_BUILD_FLAGS}"
    COMMAND sh ../build/build_rive.sh ${RIVE_CFG} ${RIVE_BUILD_FLAGS} clean
    WORKING_DIRECTORY ${RIVE_RENDERER_DIR}
    DEPENDS rive_prebuild
)

add_custom_target(rive_build ALL
    DEPENDS
        ${RIVE_RENDERER_LIB}
        ${RIVE_DECODERS_LIB}
        ${RIVE_PLS_RENDERER_LIB}
        ${RIVE_HARFBUZZ_LIB}
        ${RIVE_SHEENBIDI_LIB}
        ${RIVE_YOGA_LIB}
)

# Step 2: Import library for Viorion
add_library(rive STATIC IMPORTED GLOBAL)
set_target_properties(rive PROPERTIES
    IMPORTED_LOCATION ${RIVE_RENDERER_LIB}
    INTERFACE_INCLUDE_DIRECTORIES
        ${RIVE_INCLUDE_DIR}
        ${RIVE_DECODERS_INCLUDE_DIR}
        ${RIVE_RENDERER_INCLUDE_DIR}
)

add_library(rive_yoga STATIC IMPORTED GLOBAL)
set_target_properties(rive_yoga PROPERTIES
    IMPORTED_LOCATION ${RIVE_YOGA_LIB}
)

add_library(rive_harfbuzz STATIC IMPORTED GLOBAL)
set_target_properties(rive_harfbuzz PROPERTIES
    IMPORTED_LOCATION ${RIVE_HARFBUZZ_LIB}
)

add_library(rive_sheenbidi STATIC IMPORTED GLOBAL)
set_target_properties(rive_sheenbidi PROPERTIES
    IMPORTED_LOCATION ${RIVE_SHEENBIDI_LIB}
)

add_library(rive_decoders STATIC IMPORTED GLOBAL)
set_target_properties(rive_decoders PROPERTIES
    IMPORTED_LOCATION ${RIVE_DECODERS_LIB}
)

add_library(rive_pls STATIC IMPORTED GLOBAL)
set_target_properties(rive_pls PROPERTIES
    IMPORTED_LOCATION ${RIVE_PLS_RENDERER_LIB}
)

target_link_libraries(rive INTERFACE
    rive_yoga
    rive_harfbuzz
    rive_sheenbidi
    rive_decoders
    rive_pls
)

add_dependencies(rive rive_build)

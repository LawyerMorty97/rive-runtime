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

# This is where the rive-runtime libraries are compiled to
set(RIVE_LIB_DIR ${RIVE_RENDERER_DIR}/out/${RIVE_CFG})
if(WIN32)
    set(RIVE_LIB ${RIVE_LIB_DIR}/rive.lib)
    set(RIVE_BUILD_SCRIPT build_rive.bat)
elseif(APPLE)
    set(RIVE_LIB ${RIVE_LIB_DIR}/librive.a)
    set(RIVE_BUILD_SCRIPT build_rive.sh)
endif()

# Step 0 (windows only): Run `/build/setup_windows_dev.bat`
if(WIN32)
    add_custom_target(rive_prebuild ALL
        COMMAND ${CMAKE_COMMAND} -E echo "[Rive] Preconfiguring rive-runtime..."
        COMMAND setup_windows_dev.bat
        COMMAND powershell -ExecutionPolicy Bypass -File "${RIVE_BUILD_DIR}/setup_windows_dev.ps1"
        WORKING_DIRECTORY ${RIVE_BUILD_DIR}
    )
endif()

# Step 1: Build rive-runtime
add_custom_command(
    OUTPUT ${RIVE_LIB}
    COMMAND ${CMAKE_COMMAND} -E echo "[Rive] Building rive-runtime - config: ${RIVE_CFG}"
    COMMAND ${RIVE_BUILD_SCRIPT}
    WORKING_DIRECTORY ${RIVE_RENDERER_DIR}
    DEPENDS rive_prebuild
)

add_custom_target(rive_build ALL
    DEPENDS ${RIVE_LIB}
)

# Step 2: Import library for Viorion
add_library(rive STATIC IMPORTED GLOBAL)
set_target_properties(rive PROPERTIES
    IMPORTED_LOCATION ${RIVE_LIB}
    INTERFACE_INCLUDE_DIRECTORIES ${RIVE_INCLUDE_DIR}
)

add_dependencies(rive rive_build)

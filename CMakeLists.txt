cmake_minimum_required(VERSION 3.16)

find_package(Python3 REQUIRED)
set(PYTHON ${Python3_EXECUTABLE} CACHE STRING "Python path")
set(EMSCRIPTEN_ROOT $ENV{EMSDK}/upstream/emscripten CACHE STRING "Emscripten path")
set(CMAKE_TOOLCHAIN_FILE ${EMSCRIPTEN_ROOT}/cmake/Modules/Platform/Emscripten.cmake)
set(WEBIDL_BINDER_SCRIPT ${EMSCRIPTEN_ROOT}/tools/webidl_binder.py)

message(STATUS "PYTHON: ${PYTHON}")
message(STATUS "EMSCRIPTEN_ROOT: ${EMSCRIPTEN_ROOT}")
message(STATUS "CMAKE_TOOLCHAIN_FILE: ${CMAKE_TOOLCHAIN_FILE}")
message(STATUS "WEBIDL_BINDER_SCRIPT: ${WEBIDL_BINDER_SCRIPT}")

project(superpixel_mesh_wasm VERSION 0.1.0 LANGUAGES CXX)

set(SUPERPIXEL_MESH_IDL_FILE ${CMAKE_CURRENT_SOURCE_DIR}/superpixel_mesh.idl)
set(SUPERPIXEL_MESH_HEADER_FILE ${CMAKE_CURRENT_SOURCE_DIR}/superpixel_mesh_wrapper.h)
set(SUPERPIXEL_MESH_GLUE webidl_glue)
set(SUPERPIXEL_MESH_DIR ${CMAKE_CURRENT_SOURCE_DIR}/superpixel-mesh/)
set(SUPERPIXEL_MESH_OUTPUT SuperpixelMeshModule)
set(SUPERPIXEL_MESH_OPTIMIZATION_LEVEL "O2")

# Set the optimization level so we use the same in all targets
set(CMAKE_CXX_FLAGS_RELEASE -${SUPERPIXEL_MESH_OPTIMIZATION_LEVEL})
set(CMAKE_BUILD_TYPE Release CACHE STRING "Build Type")

# Ceres-Solver, options for build without shared libraries
option(MINIGLOG "" ON)
option(GFLAGS "" OFF)
option(SUITESPARSE "" OFF)
option(CXSPARSE "" ON)
option(LAPACK "" OFF)
option(OPENMP "" OFF)
option(SCHUR_SPECIALIZATIONS "" OFF)
option(BUILD_TESTING "" OFF)
option(BUILD_EXAMPLES "" OFF)
option(BUILD_BENCHMARKS "" OFF)
option(BUILD_SHARED_LIBS "" OFF)
set(MINIGLOG_MAX_LOG_LEVEL "0" CACHE STRING "")
add_subdirectory(ceres-solver EXCLUDE_FROM_ALL)

# For SuperPixel, we disable sample apps and unit tests
set(SUPERPIXEL_MESH_BUILD_APPS OFF)
set(SUPERPIXEL_MESH_BUILD_TESTS OFF)
add_subdirectory(superpixel-mesh EXCLUDE_FROM_ALL)

set(EMCC_GLUE_ARGS
    -c
    -${SUPERPIXEL_MESH_OPTIMIZATION_LEVEL}
    -I${CMAKE_CURRENT_BINARY_DIR}
    -I${SUPERPIXEL_MESH_DIR}/lib
    -include${SUPERPIXEL_MESH_HEADER_FILE})

# Generate glue
add_custom_command(
    OUTPUT ${SUPERPIXEL_MESH_GLUE}.cpp ${SUPERPIXEL_MESH_GLUE}.js
    BYPRODUCTS parser.out WebIDLGrammar.pkl
    COMMAND ${PYTHON} ${WEBIDL_BINDER_SCRIPT} ${SUPERPIXEL_MESH_IDL_FILE} ${SUPERPIXEL_MESH_GLUE}
    DEPENDS ${SUPERPIXEL_MESH_IDL_FILE}
    COMMENT "Generating WebIDL bindings"
    VERBATIM)
# Build glue
add_custom_command(
    OUTPUT ${SUPERPIXEL_MESH_GLUE}.o
    COMMAND emcc ${SUPERPIXEL_MESH_GLUE}.cpp ${EMCC_GLUE_ARGS} -o ${SUPERPIXEL_MESH_GLUE}.o
    DEPENDS ${SUPERPIXEL_MESH_GLUE}.cpp ${SUPERPIXEL_MESH_HEADER_FILE}
    COMMENT "Building bindings"
    VERBATIM)
add_custom_target(superpixel_bindings 
    ALL 
    DEPENDS ${SUPERPIXEL_MESH_GLUE}.js ${SUPERPIXEL_MESH_GLUE}.o)

# Targets to be exported
set(SUPERPIXEL_MESH_LIBRARIES
    $<TARGET_FILE:SuperpixelMesh::SuperpixelMesh>
    $<TARGET_FILE:Ceres::ceres>)

set(EMCC_OUTPUT_ARGS
    --post-js ${SUPERPIXEL_MESH_GLUE}.js
    -${SUPERPIXEL_MESH_OPTIMIZATION_LEVEL}
    -s EXPORT_NAME="${SUPERPIXEL_MESH_OUTPUT}"
    -s MODULARIZE=1
    -s ALLOW_MEMORY_GROWTH=1
    -s ALLOW_TABLE_GROWTH=1
    -s EXTRA_EXPORTED_RUNTIME_METHODS=["addFunction"]
	-s DISABLE_EXCEPTION_CATCHING=1
    -s USE_SDL=0
    -s ENVIRONMENT=web
    -s NO_FILESYSTEM=1)

# Generate WASM output
add_custom_command(
    OUTPUT ${SUPERPIXEL_MESH_OUTPUT}.js ${SUPERPIXEL_MESH_OUTPUT}.wasm
    COMMAND emcc ${SUPERPIXEL_MESH_GLUE}.o ${SUPERPIXEL_MESH_LIBRARIES} ${EMCC_OUTPUT_ARGS} -o ${SUPERPIXEL_MESH_OUTPUT}.js
    DEPENDS superpixel_bindings SuperpixelMesh::SuperpixelMesh Ceres::ceres
    COMMENT "Building WASM"
    VERBATIM)
add_custom_target(superpixel_wasm 
    ALL 
    DEPENDS ${SUPERPIXEL_MESH_OUTPUT}.js ${SUPERPIXEL_MESH_OUTPUT}.wasm)

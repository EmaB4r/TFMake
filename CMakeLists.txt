cmake_minimum_required(VERSION 3.12)
project(tflite_micro LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_C_STANDARD 17)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-exceptions -fno-rtti")

# FlatBuffers sources
file(GLOB_RECURSE FLATBUFFERS_SOURCES
    third_party/flatbuffers/src/*.cpp
    third_party/flatbuffers/src/*.cc
)

# KissFFT sources
file(GLOB KISSFFT_SOURCES
    third_party/kissfft/kiss_fft.c
    third_party/kissfft/tools/kiss_fftr.c
)

# TensorFlow Lite Micro sources - explicit critical files
set(TF_LITE_CRITICAL_SOURCES
    tensorflow/lite/micro/tflite_bridge/micro_error_reporter.cc
    tensorflow/lite/micro/tflite_bridge/flatbuffer_conversions_bridge.cc
    tensorflow/lite/core/c/common.cc
    tensorflow/compiler/mlir/lite/schema/schema_utils.cc
    tensorflow/compiler/mlir/lite/core/api/error_reporter.cc
)

# Additional TF sources
file(GLOB_RECURSE TF_LITE_SOURCES
    tensorflow/lite/*.cc
    tensorflow/lite/*.c
    tensorflow/lite/micro/*.cc
    tensorflow/lite/micro/*.c
    tensorflow/lite/kernels/*.cc
    tensorflow/lite/core/*.cc
    tensorflow/lite/schema/*.cc
    tensorflow/compiler/mlir/lite/schema/*.cc
)

# Signal sources
file(GLOB_RECURSE SIGNAL_SOURCES
    signal/*.cc
    signal/*.c
    signal/src/*.cc
)

# Ruy sources
file(GLOB_RECURSE RUY_SOURCES
    third_party/ruy/ruy/*.cc
    third_party/ruy/ruy/profiler/*.cc
)

# Combine all sources
set(ALL_SOURCES
    ${TF_LITE_CRITICAL_SOURCES}
    ${TF_LITE_SOURCES}
    ${SIGNAL_SOURCES}
    ${FLATBUFFERS_SOURCES}
    ${KISSFFT_SOURCES}
    ${RUY_SOURCES}
)

# Test exclusion patterns
set(TEST_FILE_PATTERNS
    "*_test.cc"
    "*test.cc"
    "*test.c"
    "hexdump_test.cc"
    "testing/*"
    "test_helper_custom_ops.cc"
    "test_helpers.cc"
)

foreach(test_pattern ${TEST_FILE_PATTERNS})
    file(GLOB_RECURSE TEST_FILES ${CMAKE_CURRENT_SOURCE_DIR}/${test_pattern})
    list(REMOVE_ITEM ALL_SOURCES ${TEST_FILES})
endforeach()

# Create main library
add_library(tflite_micro STATIC ${ALL_SOURCES})

# Include directories
target_include_directories(tflite_micro PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}/third_party/flatbuffers/include
    ${CMAKE_CURRENT_SOURCE_DIR}/third_party/ruy
    ${CMAKE_CURRENT_SOURCE_DIR}/third_party/kissfft
    ${CMAKE_CURRENT_SOURCE_DIR}/third_party/kissfft/tools
    ${CMAKE_CURRENT_SOURCE_DIR}/third_party/gemmlowp
    ${CMAKE_CURRENT_SOURCE_DIR}/tensorflow
    ${CMAKE_CURRENT_SOURCE_DIR}/tensorflow/lite
    ${CMAKE_CURRENT_SOURCE_DIR}/tensorflow/lite/micro
    ${CMAKE_CURRENT_SOURCE_DIR}/tensorflow/lite/kernels
    ${CMAKE_CURRENT_SOURCE_DIR}/tensorflow/lite/core
    ${CMAKE_CURRENT_SOURCE_DIR}/tensorflow/lite/schema
    ${CMAKE_CURRENT_SOURCE_DIR}/signal
    ${CMAKE_CURRENT_SOURCE_DIR}/tensorflow/compiler/mlir/lite/schema
)

# Compile definitions
target_compile_definitions(tflite_micro PRIVATE 
    TF_LITE_STATIC_MEMORY
    NDEBUG
    TF_LITE_USE_GLOBAL_ERROR_REPORTER=1
    TF_LITE_USE_GLOBAL_MIN=1
    TF_LITE_USE_GLOBAL_MAX=1
    TF_LITE_DISABLE_VIRTUAL_DELETE=1
    FLATBUFFERS_LOCALE_INDEPENDENT=0
    RUY_PROFILER_ENABLED=1
)

# Handle C sources
set_source_files_properties(
    ${KISSFFT_SOURCES}
    PROPERTIES LANGUAGE C
)

# Example application
add_executable(hello_world 
    examples/hello_world/hello_world_test.cc
    examples/hello_world/models/hello_world_float_model_data.cc
    examples/hello_world/models/hello_world_int8_model_data.cc
)

target_include_directories(hello_world PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/examples/hello_world
    ${CMAKE_CURRENT_SOURCE_DIR}/examples/hello_world/models
)

target_link_libraries(hello_world tflite_micro)
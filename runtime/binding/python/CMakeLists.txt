cmake_minimum_required(VERSION 3.14 FATAL_ERROR)

project(wenet VERSION 0.1)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_EXTENSIONS OFF)
# set(CMAKE_VERBOSE_MAKEFILE on)

include_directories(
${CMAKE_CURRENT_SOURCE_DIR}
${CMAKE_CURRENT_SOURCE_DIR}/kaldi  # for not changing c++ header names in kaldi source files
)

include(FetchContent)
include(ExternalProject)
set(FETCHCONTENT_QUIET off)
get_filename_component(fc_base "fc_base" REALPATH BASE_DIR "${CMAKE_CURRENT_SOURCE_DIR}")
set(FETCHCONTENT_BASE_DIR ${fc_base})

if(NOT MSVC)
  # Keep the same with openfst, -fPIC or -fpic
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++14 -pthread -fPIC")
endif()

# libtorch: here we use libtorch from python torch
execute_process(
  COMMAND python -c "import os; import torch; print(os.path.dirname(torch.__file__))"
  OUTPUT_STRIP_TRAILING_WHITESPACE
  OUTPUT_VARIABLE TORCH_DIR
)
list(APPEND CMAKE_PREFIX_PATH "${TORCH_DIR}")
find_package(Torch REQUIRED PATHS ${TORCH_DIR} NO_DEFAULT_PATH)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${TORCH_CXX_FLAGS}")
message(STATUS ${TORCH_CXX_FLAGS})
message(STATUS "${CMAKE_CXX_FLAGS}")

# third_party: gflags
FetchContent_Declare(gflags
  URL      https://github.com/gflags/gflags/archive/v2.2.1.zip
  URL_HASH SHA256=4e44b69e709c826734dbbbd5208f61888a2faf63f239d73d8ba0011b2dccc97a
)
FetchContent_MakeAvailable(gflags)
include_directories(${gflags_BINARY_DIR}/include)

# third_party: glog
FetchContent_Declare(glog
  URL      https://github.com/google/glog/archive/v0.4.0.zip
  URL_HASH SHA256=9e1b54eb2782f53cd8af107ecf08d2ab64b8d0dc2b7f5594472f3bd63ca85cdc
)
FetchContent_MakeAvailable(glog)
include_directories(${glog_SOURCE_DIR}/src ${glog_BINARY_DIR})

# utils
add_library(utils STATIC
  utils/string.cc
  utils/utils.cc
)

# The original openfst uses GNU Build System to run configure and build.
# So, we use "OpenFST port for Windows" to build openfst with cmake in Windows.
# Openfst is compiled with glog/gflags to avoid log and flag conflicts with log and flags in wenet/libtorch.
# To build openfst with gflags and glog, we comment out some vars of {flags, log}.h and flags.cc.
if(NOT MSVC)
  set(openfst_SOURCE_DIR ${fc_base}/openfst-src CACHE PATH "OpenFST source directory")
  set(openfst_BINARY_DIR ${fc_base}/openfst-build CACHE PATH "OpenFST build directory")
  set(openfst_PREFIX_DIR ${fc_base}/openfst-subbuild/openfst-populate-prefix CACHE PATH "OpenFST prefix directory")
  ExternalProject_Add(openfst
    URL               https://github.com/mjansche/openfst/archive/1.6.5.zip
    URL_HASH          SHA256=b720357a464f42e181d7e33f60867b54044007f50baedc8f4458a3926f4a5a78
    SOURCE_DIR        ${openfst_SOURCE_DIR}
    BINARY_DIR        ${openfst_BINARY_DIR}
    CONFIGURE_COMMAND ${openfst_SOURCE_DIR}/configure --prefix=${openfst_PREFIX_DIR}
                        "CPPFLAGS=-I${gflags_BINARY_DIR}/include -I${glog_SOURCE_DIR}/src -I${glog_BINARY_DIR} ${TORCH_CXX_FLAGS}"
                        "LDFLAGS=-L${gflags_BINARY_DIR} -L${glog_BINARY_DIR}"
                        "LIBS=-lgflags_nothreads -lglog -lpthread"
    COMMAND           ${CMAKE_COMMAND} -E copy_directory ${CMAKE_CURRENT_SOURCE_DIR}/patch/openfst ${openfst_SOURCE_DIR}
    BUILD_COMMAND     make -j 4
  )
  add_dependencies(openfst gflags glog)
  add_dependencies(utils openfst)
  link_directories(${openfst_PREFIX_DIR}/lib)
  target_link_libraries(utils PUBLIC fst dl)
else()
  add_compile_options(/W0 /wd4244 /wd4267)
  add_compile_options("$<$<CXX_COMPILER_ID:MSVC>:/utf-8>")
  add_definitions(-DBOOST_ALL_DYN_LINK -DBOOST_ALL_NO_LIB)
  file(GLOB TORCH_DLLS "${TORCH_INSTALL_PREFIX}/lib/*.dll")
  file(COPY ${TORCH_DLLS} DESTINATION ${CMAKE_BINARY_DIR})
  FetchContent_Declare(openfst
    URL      https://github.com/kkm000/openfst/archive/refs/tags/win/1.6.5.1.tar.gz
    URL_HASH SHA256=02c49b559c3976a536876063369efc0e41ab374be1035918036474343877046e
  )
  FetchContent_MakeAvailable(openfst)
  execute_process(COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_CURRENT_SOURCE_DIR}/patch/openfst ${openfst_SOURCE_DIR})
  add_dependencies(fst gflags glog)
  target_link_libraries(fst PUBLIC gflags_nothreads_static glog)
  target_link_libraries(utils PUBLIC fst)
endif()
include_directories(${openfst_SOURCE_DIR}/src/include)


FetchContent_Declare(pybind11
  URL      https://github.com/pybind/pybind11/archive/refs/tags/v2.9.2.zip
  URL_HASH SHA256=d1646e6f70d8a3acb2ddd85ce1ed543b5dd579c68b8fb8e9638282af20edead8
)
FetchContent_MakeAvailable(pybind11)
add_subdirectory(${pybind11_SOURCE_DIR})

# frontend
add_library(frontend STATIC
  frontend/feature_pipeline.cc
  frontend/fft.cc
)
target_link_libraries(frontend PUBLIC utils)

# kaldi: wfst based decoder
add_subdirectory(kaldi)

# decoder
add_library(decoder STATIC
  decoder/asr_decoder.cc
  decoder/asr_model.cc
  decoder/context_graph.cc
  decoder/ctc_prefix_beam_search.cc
  decoder/ctc_wfst_beam_search.cc
  decoder/ctc_endpoint.cc
  decoder/torch_asr_model.cc
)
target_link_libraries(decoder PUBLIC ${TORCH_LIBRARIES} kaldi-decoder utils post_processor)

# post_processor
add_library(post_processor STATIC
  post_processor/post_processor.cc
)
target_link_libraries(post_processor PUBLIC utils)

add_library(wenet_api SHARED api/wenet_api.cc)
target_link_libraries(wenet_api PUBLIC decoder frontend)

# wenet api
pybind11_add_module(_wenet cpp/binding.cc)
target_link_libraries(_wenet PRIVATE wenet_api)

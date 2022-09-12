cmake_minimum_required(VERSION 3.10.2)
project("openfst")

include(ExternalProject)
set(CMAKE_VERBOSE_MAKEFILE on)
set(ANDROID_CMAKE_ARGS
  -DBUILD_TESTING=OFF
  -DBUILD_SHARED_LIBS=OFF
  -DCMAKE_BUILD_TYPE=Release
  -DCMAKE_MAKE_PROGRAM=${CMAKE_MAKE_PROGRAM}
  -DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE}
  -DANDROID_ABI=${ANDROID_ABI}
  -DANDROID_NATIVE_API_LEVEL=${ANDROID_NATIVE_API_LEVEL}
)

ExternalProject_Add(gflags
  URL              https://github.com/gflags/gflags/archive/v2.2.0.zip
  URL_HASH         SHA256=99f9e8b63ea53e4b23f0fd0f68a6f1e397d5512be36716c17cc75966a90f0d57
  BUILD_ALWAYS     true
  CMAKE_ARGS       ${ANDROID_CMAKE_ARGS}
  INSTALL_COMMAND  ""
)
ExternalProject_Get_Property(gflags BINARY_DIR)
include_directories(${BINARY_DIR}/include)
link_directories(${BINARY_DIR}/lib)
set(gflags_BINARY_DIR ${BINARY_DIR})

ExternalProject_Add(glog
  URL              https://github.com/google/glog/archive/refs/tags/v0.5.0.zip
  URL_HASH         SHA256=21bc744fb7f2fa701ee8db339ded7dce4f975d0d55837a97be7d46e8382dea5a
  BUILD_ALWAYS     true
  CMAKE_ARGS       ${ANDROID_CMAKE_ARGS}
  INSTALL_COMMAND  ""
)
ExternalProject_Get_Property(glog SOURCE_DIR BINARY_DIR)
include_directories(${SOURCE_DIR}/src ${BINARY_DIR})
link_directories(${BINARY_DIR})
set(glog_SOURCE_DIR ${SOURCE_DIR})
set(glog_BINARY_DIR ${BINARY_DIR})

set(openfst_SOURCE_DIR ${CMAKE_BINARY_DIR}/openfst-src)
ExternalProject_Add(openfst
  URL               https://github.com/mjansche/openfst/archive/1.6.5.zip
  URL_HASH          SHA256=b720357a464f42e181d7e33f60867b54044007f50baedc8f4458a3926f4a5a78
  SOURCE_DIR        ${openfst_SOURCE_DIR}
  CONFIGURE_COMMAND PATH=${CMAKE_PREFIX_PATH}/bin:$ENV{PATH} ${openfst_SOURCE_DIR}/configure
                      --disable-bin --enable-static --host=${CMAKE_LIBRARY_ARCHITECTURE}
                      "CXX=${ANDROID_CXX_COMPILER} -target ${CMAKE_LIBRARY_ARCHITECTURE}${ANDROID_NATIVE_API_LEVEL}"
                      "CPPFLAGS=-I${gflags_BINARY_DIR}/include -I${glog_SOURCE_DIR}/src -I${glog_BINARY_DIR} -fPIC"
                      "LIBS=${gflags_BINARY_DIR}/lib/libgflags_nothreads.a ${glog_BINARY_DIR}/libglog.a"
  COMMAND           cp ${CMAKE_SOURCE_DIR}/patch/log.h ${openfst_SOURCE_DIR}/src/include/fst
  COMMAND           cp ${CMAKE_SOURCE_DIR}/patch/flags.h ${openfst_SOURCE_DIR}/src/include/fst
  COMMAND           cp ${CMAKE_SOURCE_DIR}/patch/flags.cc ${openfst_SOURCE_DIR}/src/lib
  BUILD_COMMAND     PATH=${CMAKE_PREFIX_PATH}/bin:$ENV{PATH} make -j 4
  INSTALL_COMMAND   ""
)
ExternalProject_Get_Property(openfst BINARY_DIR)
include_directories(${openfst_SOURCE_DIR}/src/include)
link_directories(${BINARY_DIR}/src/lib/.libs)
add_dependencies(openfst gflags glog)

link_libraries(log gflags_nothreads glog fst)
add_library(native-lib SHARED native-lib.cpp)
add_dependencies(native-lib openfst)

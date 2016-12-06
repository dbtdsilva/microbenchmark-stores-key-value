project(microbenchmarks)
cmake_minimum_required(VERSION 2.8 FATAL_ERROR)

if (NOT CMAKE_SIZEOF_VOID_P EQUAL 8)
    message(FATAL_ERROR "Doesn't support non 64-bit platforms")
endif()

include(FindThreads)
include(CheckTypeSize)
include(CheckIncludeFiles)
include(CheckCXXCompilerFlag)
include(ExternalProject)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

CHECK_CXX_COMPILER_FLAG("-std=c++11" CXX11)
if (NOT CXX11)
    message(FATAL_ERROR "C++ compiler doesn't support C++11")
endif()

include_directories(${microbenchmarks_SOURCE_DIR})

set(lmdb_PREFIX ${CMAKE_CURRENT_BINARY_DIR}/external/lmdb)
ExternalProject_Add(
    lmdb
    PREFIX ${lmdb_PREFIX}
    URL "https://github.com/LMDB/lmdb/archive/LMDB_0.9.18.tar.gz"
    URL_MD5 "8b7eeb8a6c30b2763581de455d10441b"
    CONFIGURE_COMMAND ""
    BUILD_COMMAND $(MAKE) -C ${lmdb_PREFIX}/src/lmdb/libraries/liblmdb/
    INSTALL_COMMAND ""
    BUILD_IN_SOURCE 1
    LOG_UPDATE ON
    LOG_BUILD ON
    LOG_INSTALL ON
)
set(lmdb_INCLUDE_DIRS ${lmdb_PREFIX})
set(lmdb_LIBRARIES ${lmdb_PREFIX}/src/lmdb/libraries/liblmdb/liblmdb.a)
include_directories(${lmdb_INCLUDE_DIRS}/src/lmdb/libraries/liblmdb)
message(STATUS "Using LMDB from ${lmdb_LIBRARIES}")


set(LINKLIBS
    ${lmdb_LIBRARIES}
    ${microbenchmarks_SOURCE_DIR}/leveldb-1.19/out-static/libleveldb.a
    ${microbenchmarks_SOURCE_DIR}/leveldb-1.19/out-static/util/testutil.o
)

include_directories(${microbenchmarks_SOURCE_DIR}/leveldb-1.19)
include_directories(${microbenchmarks_SOURCE_DIR}/leveldb-1.19/include)

add_executable(
    benchmark
    db_bench_mdb.cc
    ${microbenchmarks_SOURCE_DIR}/leveldb-1.19
)
target_link_libraries(benchmark ${LINKLIBS} -fno-builtin-memcmp -pthread -DOS_LINUX -DLEVELDB_PLATFORM_POSIX 
    -DLEVELDB_ATOMIC_PRESENT -DSNAPPY -O2 -DNDEBUG -fPIC -lsnappy)
add_dependencies(benchmark lmdb)
set_target_properties(benchmark PROPERTIES COMPILE_FLAGS "-std=c++14 -O3")


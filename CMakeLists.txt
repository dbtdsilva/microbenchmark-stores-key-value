project(microbenchmarks)
cmake_minimum_required(VERSION 2.8 FATAL_ERROR)

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

find_package(ZLIB)
if (NOT ZLIB_FOUND)
    message(FATAL_ERROR "Couldn't find zlib library")
endif()
include_directories(${ZLIB_INCLUDE_DIRS})

include_directories(${microbenchmarks_SOURCE_DIR})

### LevelDB Compilation, it offers the benchmark service
set(leveldb_PREFIX ${CMAKE_CURRENT_BINARY_DIR}/external/leveldb)
ExternalProject_Add(
    leveldb
    PREFIX ${leveldb_PREFIX}
    URL "https://github.com/google/leveldb/archive/v1.19.tar.gz"
    URL_MD5 "6c201409cce6b711f46d68e0f4b1090a"
    CONFIGURE_COMMAND ""
    BUILD_COMMAND $(MAKE)
    INSTALL_COMMAND ""
    BUILD_IN_SOURCE 1
    LOG_UPDATE ON
    LOG_BUILD ON
    LOG_INSTALL ON
)
set(leveldb_INCLUDE_DIRS 
    ${leveldb_PREFIX}/src/leveldb 
    ${leveldb_PREFIX}/src/leveldb/include)
set(leveldb_LIBRARIES 
    ${leveldb_PREFIX}/src/leveldb/out-static/libleveldb.a
    ${leveldb_PREFIX}/src/leveldb/out-static/libmemenv.a
    ${leveldb_PREFIX}/src/leveldb/out-static/util/testutil.o)
include_directories(${leveldb_INCLUDE_DIRS})

### LMDB Compilation
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
set(lmdb_INCLUDE_DIRS ${lmdb_PREFIX}/src/lmdb/libraries/liblmdb)
set(lmdb_LIBRARIES ${lmdb_PREFIX}/src/lmdb/libraries/liblmdb/liblmdb.a)
include_directories(${lmdb_INCLUDE_DIRS})

### BerkeleyDB Compilation
set(bdb_PREFIX ${CMAKE_CURRENT_BINARY_DIR}/external/bdb)
ExternalProject_Add(
    bdb
    PREFIX ${bdb_PREFIX}
    URL "http://download.oracle.com/berkeley-db/db-6.2.23.tar.gz"
    URL_MD5 "9da126afb937a48171ff1e569ce67cf1"
    CONFIGURE_COMMAND CXX=${CMAKE_CXX_COMPILER} CC=${CMAKE_C_COMPILER} ${bdb_PREFIX}/src/bdb/dist/configure --prefix=${bdb_PREFIX}/lib -enable-shared=no --enable-cxx
    BUILD_COMMAND $(MAKE)
    INSTALL_COMMAND $(MAKE) install
    BUILD_IN_SOURCE 1
    LOG_UPDATE ON
    LOG_BUILD ON
    LOG_INSTALL ON
)
set(bdb_INCLUDE_DIRS ${bdb_PREFIX})
set(bdb_LIBRARIES ${bdb_PREFIX}/lib/lib/libdb.a)
include_directories(${bdb_INCLUDE_DIRS}/lib/include)

### Kyoto Compilation
set(kyoto_PREFIX ${CMAKE_CURRENT_BINARY_DIR}/external/kyoto)
ExternalProject_Add(
    kyoto
    PREFIX ${kyoto_PREFIX}
    URL "http://fallabs.com/kyotocabinet/pkg/kyotocabinet-1.2.76.tar.gz"
    URL_MD5 "b925d369d7ed4a37cf0013409046738c"
    CONFIGURE_COMMAND CXX=g++-5 CC=gcc-5 ${kyoto_PREFIX}/src/kyoto/configure --prefix=${kyoto_PREFIX}/lib 
    BUILD_COMMAND $(MAKE)
    INSTALL_COMMAND $(MAKE) install
    BUILD_IN_SOURCE 1
    LOG_UPDATE ON
    LOG_BUILD ON
    LOG_INSTALL ON
)
set(kyoto_INCLUDE_DIRS ${kyoto_PREFIX}/lib/include)
set(kyoto_LIBRARIES ${kyoto_PREFIX}/lib/lib/libkyotocabinet.a)
include_directories(${kyoto_INCLUDE_DIRS})

### Linkage and executable
## General definitions
add_definitions(-DOS_LINUX -DLEVELDB_PLATFORM_POSIX -DLEVELDB_ATOMIC_PRESENT -DSNAPPY -DNDEBUG -O3)
## LMDB
set(LMDB_LINKLIBS
    ${lmdb_LIBRARIES}
    ${leveldb_LIBRARIES}
)

add_executable(
    lmdb_benchmark
    db_bench_mdb.cc
)
target_link_libraries(lmdb_benchmark ${LMDB_LINKLIBS} -fno-builtin-memcmp -pthread -fPIC -lsnappy)
add_dependencies(lmdb_benchmark lmdb)
set_target_properties(lmdb_benchmark PROPERTIES COMPILE_FLAGS "-std=c++14 -O3")

## BDB
set(BDB_LIBRARIES
    ${leveldb_LIBRARIES}
    ${bdb_LIBRARIES}
)
add_executable(
    bdb_benchmark
    db_bench_bdb.cc
)
target_link_libraries(bdb_benchmark ${BDB_LIBRARIES} -fno-builtin-memcmp -pthread -fPIC -lsnappy)
add_dependencies(bdb_benchmark bdb)
set_target_properties(bdb_benchmark PROPERTIES COMPILE_FLAGS "-std=c++14 -O3")

## Koyot Cabinet
set(KYOTO_LIBRARIES
    ${leveldb_LIBRARIES}
    ${kyoto_LIBRARIES}
)
add_executable(
    kyoto_benchmark
    db_bench_tree_db.cc
)
target_link_libraries(kyoto_benchmark ${KYOTO_LIBRARIES} -fno-builtin-memcmp -pthread -fPIC -lsnappy -lz)
add_dependencies(kyoto_benchmark kyoto)
set_target_properties(kyoto_benchmark PROPERTIES COMPILE_FLAGS "-std=c++14 -O3")

## LevelDB
set(LEVELDB_LIBRARIES
    ${leveldb_LIBRARIES}
)
add_executable(
    leveldb_benchmark
    db_bench_leveldb.cc
)
target_link_libraries(leveldb_benchmark ${LEVELDB_LIBRARIES} -fno-builtin-memcmp -pthread -fPIC -lsnappy)
add_dependencies(leveldb_benchmark leveldb)
set_target_properties(leveldb_benchmark PROPERTIES COMPILE_FLAGS "-std=c++14 -O3")

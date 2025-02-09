# It's version of cmake in our oldest supported distro CentOS 7: https://repology.org/project/cmake/versions
cmake_minimum_required (VERSION 2.8.12)

set(FASTNETMON_LIBRARIES_GLOBAL_PATH "/opt/fastnetmon-community/libraries")

project(FastNetMon)

include(GNUInstallDirs)

include(CheckCXXCompilerFlag)
include(CheckLibraryExists)

# Enable it and fix all warnings
# add_definitions ("-Wall")

set (FASTNETMON_VERSION_MAJOR 1)
set (FASTNETMON_VERSION_MINOR 2)
set (FASTNETMON_VERSION_PATCH 3)

set(HIREDIS_CUSTOM_INSTALL_PATH "${FASTNETMON_LIBRARIES_GLOBAL_PATH}/libhiredis_0_13")
set(LOG4CPP_CUSTOM_INSTALL_PATH "${FASTNETMON_LIBRARIES_GLOBAL_PATH}/log4cpp1.1.3")
set(LIBPCAP_CUSTOM_INSTALL_PATH "${FASTNETMON_LIBRARIES_GLOBAL_PATH}/libpcap_1.7.4")
set(MONGO_C_CUSTOM_INSTALL_PATH "${FASTNETMON_LIBRARIES_GLOBAL_PATH}/mongo_c_driver_1_16_1")
set(CAPNP_CUSTOM_INSTALL_PATH "${FASTNETMON_LIBRARIES_GLOBAL_PATH}/capnproto_0_8_0")
set(OPENSSL_CUSTOM_INSTALL_PATH "${FASTNETMON_LIBRARIES_GLOBAL_PATH}/openssl_1_0_2d")
set(GRPC_CUSTOM_INSTALL_PATH "${FASTNETMON_LIBRARIES_GLOBAL_PATH}/grpc_1_30_2")

set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} --std=c++2a")
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} --std=c++2a")

# With this flag we can enable GoBGP build via console: cmake .. -DENABLE_GOBGP_SUPPORT=ON
option(ENABLE_GOBGP_SUPPORT "Enable GoBGP support build" ON)

# On some platforms like Fedora we need to explicitly link with libabseil / libabsl.
# We do not use it directly but gRPC uses it for libgpr and we need to link with absl explicitly to avoid linker errors like this:
# libgpr.so: undefined reference to symbol '_ZN4absl12lts_202103245Mutex4LockEv'
# /usr/lib64/libabsl_synchronization.so.2103.0.1: error adding symbols: DSO missing from command line
option(LINK_WITH_ABSL "Enable optonal linking with ABSL" OFF)

option(DO_NOT_USE_SYSTEM_LIBRARIES_FOR_BUILD "Disables use of libraries from system path" ON)

if (DO_NOT_USE_SYSTEM_LIBRARIES_FOR_BUILD)
    # We need to avoid using system path for libraries and includes search because we ship specific versions of our own libraries in package
    # And we need to avoid implit fallbacks to system libraries as it will break dependencies
    set(DISABLE_DEFAULT_PATH_SEARCH_VAR "NO_DEFAULT_PATH")
else ()
    # Disable this logic and allow any paths
    set(DISABLE_DEFAULT_PATH_SEARCH_VAR "")
endif()

option(ENABLE_CUSTOM_BOOST_BUILD "Enable build with custom Boost" ON)
if (ENABLE_CUSTOM_BOOST_BUILD)
    message(STATUS "Build with custom Boost")

    set(Boost_NO_SYSTEM_PATHS ON)

    set(BOOST_INCLUDEDIR "${FASTNETMON_LIBRARIES_GLOBAL_PATH}/boost_1_78_0")
    set(BOOST_LIBRARYDIR "${FASTNETMON_LIBRARIES_GLOBAL_PATH}/boost_1_78_0/stage/lib/")

    SET(Boost_DIR "${FASTNETMON_LIBRARIES_GLOBAL_PATH}/boost_1_78_0/stage/lib/cmake/Boost-1.78.0/")

    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -Wno-deprecated-declarations")

    set(ICU_CUSTOM_INSTALL_PATH "${FASTNETMON_LIBRARIES_GLOBAL_PATH}/libicu_65_1")

    set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_RPATH};${FASTNETMON_LIBRARIES_GLOBAL_PATH}/boost_1_78_0/stage/lib;${ICU_CUSTOM_INSTALL_PATH}/lib;${FASTNETMON_LIBRARIES_GLOBAL_PATH}/gcc1210/lib64")

    # For custom Boost we need to build libicu library
    find_library(ICU_LIBRARY_UC_PATH NAMES icuuc PATHS "${ICU_CUSTOM_INSTALL_PATH}/lib" ${DISABLE_DEFAULT_PATH_SEARCH_VAR})
    find_library(ICU_LIBRARY_DATA_PATH NAMES icudata PATHS "${ICU_CUSTOM_INSTALL_PATH}/lib" ${DISABLE_DEFAULT_PATH_SEARCH_VAR})
    find_library(ICU_LIBRARY_I18N_PATH NAMES icui18n PATHS "${ICU_CUSTOM_INSTALL_PATH}/lib" ${DISABLE_DEFAULT_PATH_SEARCH_VAR})

    if (ICU_LIBRARY_UC_PATH AND ICU_LIBRARY_DATA_PATH AND ICU_LIBRARY_I18N_PATH)
        message(STATUS "Found libicu")
    else()
        message(FATAL_ERROR "Could not find icu libraries ${ICU_LIBRARY_UC_PATH} ${ICU_LIBRARY_DATA_PATH} ${ICU_LIBRARY_I18N_PATH}")
    endif()

endif()

# We use hardcoded RPATH for our libraries only when we compile against our custom libraries
if (DO_NOT_USE_SYSTEM_LIBRARIES_FOR_BUILD) 

    # Specify full RPATH for build tree
    SET(CMAKE_SKIP_BUILD_RPATH  FALSE)

    # Create builds in current folder with install RPATH
    SET(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)

    SET(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_RPATH};${FASTNETMON_LIBRARIES_GLOBAL_PATH}/libhiredis_0_13/lib;${FASTNETMON_LIBRARIES_GLOBAL_PATH}/log4cpp1.1.3/lib;${FASTNETMON_LIBRARIES_GLOBAL_PATH}/json-c-0.13/lib;${MONGO_C_CUSTOM_INSTALL_PATH}/lib;${MONGO_C_CUSTOM_INSTALL_PATH}/lib64;${GRPC_CUSTOM_INSTALL_PATH}/lib;${FASTNETMON_LIBRARIES_GLOBAL_PATH}/protobuf_3.11.4/lib;${CAPNP_CUSTOM_INSTALL_PATH}/lib;${OPENSSL_CUSTOM_INSTALL_PATH}/lib")

else()
   # We do not need any RPATH alterations when we want to link with system libraries (i.e. upstream builds for Debian or RedHat family)
endif()

message(STATUS "C++ compilation flags: ${CMAKE_CXX_FLAGS_RELEASE}")
message(STATUS "C++ debug compilation flags: ${CMAKE_CXX_FLAGS_DEBUG}")

set(FASTNETMON_PROFILER OFF)

set(FASTNETMON_PROFILE_FLAGS "-g -pg")

# set(CMAKE_BUILD_TYPE DEBUG)

if (NOT CMAKE_BUILD_TYPE) 
    message(STATUS "Setting build type to Release as none was specified.")
    set(CMAKE_BUILD_TYPE Release)
endif()

if (FASTNETMON_PROFILER) 
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${FASTNETMON_PROFILE_FLAGS}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${FASTNETMON_PROFILE_FLAGS}")
    set(CMAKE_EXE_LINKER_FLAGS  "${CMAKE_EXE_LINKER_FLAGS} ${FASTNETMON_PROFILE_FLAGS}")
endif()

execute_process(COMMAND sh -c ". /etc/os-release; echo $ID" OUTPUT_VARIABLE OS_ID ERROR_QUIET)

### Executables definition 

# Main tool
add_executable(fastnetmon fastnetmon.cpp)

# Get last commit hash
execute_process(COMMAND git rev-list HEAD COMMAND head -n 1 OUTPUT_VARIABLE GIT_LAST_COMMIT_HASH OUTPUT_STRIP_TRAILING_WHITESPACE)

set(FASTNETMON_APPLICATION_VERSION "${FASTNETMON_VERSION_MAJOR}.${FASTNETMON_VERSION_MINOR}.${FASTNETMON_VERSION_PATCH} master git-${GIT_LAST_COMMIT_HASH}")
configure_file(fast_platform.h.template "${PROJECT_SOURCE_DIR}/fast_platform.hpp")

# Use new Memory Model Aware Atomic Operations
# You could enable it using: cmake .. -DUSE_NEW_ATOMIC_BUILTINS=ON
# We use it for exotic platforms where we have no specific functions in atomics library: https://salsa.debian.org/debian/fastnetmon/-/blob/master/debian/rules#L11
if (USE_NEW_ATOMIC_BUILTINS)
    message(STATUS "Will use new memory model aware atomic builtins")
    add_definitions(-DUSE_NEW_ATOMIC_BUILTINS) 
endif() 

# Be default we do not link with it but we need it on platforms without native support for atomic operations
# On these platforms we will use logic emulated by libatomic
set(LINK_WITH_ATOMIC_LIBRARY OFF)

CHECK_CXX_SOURCE_COMPILES("
#include <cstdint>
int main() {
    uint64_t x = 1;
    __atomic_add_fetch(&x, 0, __ATOMIC_RELAXED);
    return x;
}
" HAVE__ATOMIC_ADD_FETCH)

if (HAVE__ATOMIC_ADD_FETCH)
    message(STATUS "We have __atomic_add_fetch on this platform")
else()
    message(STATUS "We have no __atomic_add_fetch, will try linking with libatomic")

    check_library_exists(atomic __atomic_add_fetch_8 "" HAVE_LIBATOMIC)

    if (HAVE_LIBATOMIC)
        message(STATUS "Linked with atomic library")
        set(LINK_WITH_ATOMIC_LIBRARY ON)
    else()
        message(STATUS "We have no support for __atomic_add_fetch in atomic library, skip linking")
    endif()
endif()

CHECK_CXX_SOURCE_COMPILES("
#include <cstdint>
int main() {
    uint64_t x = 1;
    __sync_fetch_and_add(&x, 1);
    return x;
}
" HAVE__SYNC_FETCH_AND_ADD)

if (HAVE__SYNC_FETCH_AND_ADD) 
    message(STATUS "We have __sync_fetch_and_add on this platform")
else()
    # We know that it happens for mipsel platform due to https://reviews.llvm.org/D45691
    message(STATUS "We have no __sync_fetch_and_add on this platform, will try linking with libatomic")
    
    check_library_exists(atomic __sync_fetch_and_add_8 "" HAVE_LIBATOMIC_SYNC_FETCH_AND_ADD)

    if (HAVE_LIBATOMIC_SYNC_FETCH_AND_ADD)
        message(STATUS "Linked with atomic library")
        set(LINK_WITH_ATOMIC_LIBRARY ON)
    else()
        message(STATUS "We have no support for __sync_fetch_and_add in atomic library, skip linking")
    endif()
endif()

option(ENABLE_NETMAP_SUPPORT "Enable Netmap support" OFF)

CHECK_CXX_SOURCE_COMPILES("
int main() {
    __atomic_thread_fence(__ATOMIC_RELEASE);
    __atomic_thread_fence(__ATOMIC_ACQUIRE);
    return 0;
}
" HAVE_ATOMIC_THREAD_FENCE)

# If we do not have it then we need to disable it
if (NOT HAVE_ATOMIC_THREAD_FENCE) 
    set(ENABLE_NETMAP_SUPPORT OFF)
    message(STATUS "Your system does not support __atomic_thread_fence, disabled Netmap plugin support")
endif()

if (ENABLE_NETMAP_SUPPORT)
    message(STATUS "We will build Netmap support for you")
    add_definitions(-DNETMAP_PLUGIN)
endif()

# It's enabled by default but can be disabled using: 
# cmake .. -DENABLE_CAPNP_SUPPORT=OFF
option(ENABLE_CAPNP_SUPPORT "Enable Cap'N'Proto support build" ON)

if (ENABLE_CAPNP_SUPPORT)

message(STATUS "We will build Cap'N'Proto support")

find_program(CAPNP_BINARY capnp PATHS "${CAPNP_CUSTOM_INSTALL_PATH}/bin" ${DISABLE_DEFAULT_PATH_SEARCH_VAR})

if (CAPNP_BINARY)
    message(STATUS "Found capnp compiler: ${CAPNP_BINARY}")
else()
    message(FATAL_ERROR "Can't find capnp compiler")
endif()


# Generate capnp bindings
ADD_CUSTOM_COMMAND(
    OUTPUT ${PROJECT_SOURCE_DIR}/simple_packet_capnp/simple_packet.capnp.c++
    DEPENDS ${PROJECT_SOURCE_DIR}/simple_packet_capnp/simple_packet.capnp
    COMMAND "LD_LIBRARY_PATH=$ENV{LD_LIBRARY_PATH}" "PATH=$ENV{PATH}:${CAPNP_CUSTOM_INSTALL_PATH}/bin" ${CAPNP_BINARY} compile --output c++:${PROJECT_SOURCE_DIR}/simple_packet_capnp --src-prefix=${PROJECT_SOURCE_DIR}/simple_packet_capnp ${PROJECT_SOURCE_DIR}/simple_packet_capnp/simple_packet.capnp
    COMMENT "Build Cap'n'Proto binding for C++"
)

add_library(simple_packet_capnp STATIC simple_packet_capnp/simple_packet.capnp.c++)

endif()

# Our LPM library
add_library(patricia STATIC libpatricia/patricia.cpp)

# Graphite metrics
add_library(graphite_metrics STATIC metrics/graphite.cpp)
target_link_libraries(fastnetmon graphite_metrics)

# InfluxDB metrics
add_library(influxdb_metrics STATIC metrics/influxdb.cpp)
target_link_libraries(fastnetmon influxdb_metrics)

add_library(fastnetmon_pcap_format STATIC fastnetmon_pcap_format.cpp)

# Our tools library
add_library(fast_library STATIC fast_library.cpp)

# Our ipfix database library
add_library(ipfix_rfc STATIC ipfix_rfc.cpp)

add_library(bgp_protocol STATIC bgp_protocol.cpp)

# Our logic library
add_library(fastnetmon_logic STATIC fastnetmon_logic.cpp)

CHECK_CXX_SOURCE_COMPILES("
#include <linux/if_packet.h>
int main() {
    return TPACKET_V3;
}
" HAVE_TPACKET_V3)


if (${HAVE_TPACKET_V3})
    message(STATUS "Your system has support for AF_PACKET v3")
    set (ENABLE_AFPACKET_SUPPORT ON)
else()
    message(STATUS "Your system does not support AF_PACKET v3, disabled it")
endif()

# -DENABLE_AFPACKET_SUPPORT=ON ..
if (ENABLE_AFPACKET_SUPPORT)
   add_definitions(-DFASTNETMON_ENABLE_AFPACKET) 
   add_library(afpacket_plugin STATIC afpacket_plugin/afpacket_collector.cpp)
endif()

# Library with data types for parsing network structures
add_library(network_data_structures STATIC network_data_structures.cpp)

# Our new parser for parsing traffic up to L4
add_library(simple_packet_parser_ng STATIC simple_packet_parser_ng.cpp)
target_link_libraries(simple_packet_parser_ng network_data_structures)

# Our own sFlow parser library
set_source_files_properties(libsflow/libsflow.cpp PROPERTIES COMPILE_FLAGS -pedantic)
add_library(libsflow STATIC libsflow/libsflow.cpp)

# sFlow plugin
add_library(sflow_plugin STATIC sflow_plugin/sflow_collector.cpp)

# Link sFlow plugin with new traffic parser
target_link_libraries(sflow_plugin simple_packet_parser_ng)

# Link sFlow plugin with libsflow
target_link_libraries(sflow_plugin libsflow)  

# netflow library
add_library(netflow STATIC netflow_plugin/netflow.cpp)

# netflow plugin
add_library(netflow_plugin STATIC netflow_plugin/netflow_collector.cpp)
target_link_libraries(netflow_plugin ipfix_rfc netflow)

# pcap plugin
add_library(pcap_plugin STATIC pcap_plugin/pcap_collector.cpp)
target_link_libraries(pcap_plugin pcap)

find_package(Threads)

add_library(exabgp_action STATIC actions/exabgp_action.cpp)

if (LINK_WITH_ABSL)
    find_package(absl REQUIRED)

    # TODO: check that we actually found it. Otherwise trigger fatal erorr
endif()

if (ENABLE_GOBGP_SUPPORT)
    set(PROTOCOL_BUFFERS_CUSTOM_INSTALL_PATH "${FASTNETMON_LIBRARIES_GLOBAL_PATH}/protobuf_3.11.4")

    add_definitions(-DENABLE_GOBGP)
    add_library(gobgp_action STATIC actions/gobgp_action.cpp)

    find_path(GRPC_INCLUDES_FOLDER NAMES grpc/grpc.h PATHS "${GRPC_CUSTOM_INSTALL_PATH}/include" ${DISABLE_DEFAULT_PATH_SEARCH_VAR})
    find_library(GRPC_LIBRARY_GRPC_PATH NAMES grpc PATHS "${GRPC_CUSTOM_INSTALL_PATH}/lib" ${DISABLE_DEFAULT_PATH_SEARCH_VAR})
    find_library(GRPC_LIBRARY_GPR_PATH NAMES gpr PATHS "${GRPC_CUSTOM_INSTALL_PATH}/lib" ${DISABLE_DEFAULT_PATH_SEARCH_VAR})
    find_library(GRPC_LIBRARY_GRPC_CPP_PATH NAMES grpc++ PATHS "${GRPC_CUSTOM_INSTALL_PATH}/lib" ${DISABLE_DEFAULT_PATH_SEARCH_VAR}) 

    if (GRPC_INCLUDES_FOLDER AND GRPC_LIBRARY_GRPC_PATH AND GRPC_LIBRARY_GPR_PATH AND GRPC_LIBRARY_GRPC_CPP_PATH)
        include_directories(${GRPC_INCLUDES_FOLDER})
        target_link_libraries(gobgp_action ${GRPC_LIBRARY_GRPC_PATH})
        target_link_libraries(gobgp_action ${GRPC_LIBRARY_GPR_PATH})
        target_link_libraries(gobgp_action ${GRPC_LIBRARY_GRPC_CPP_PATH})
        
	    message(STATUS "Found gRPC library: ${GRPC_LIBRARY_GRPC_PATH} ${GRPC_LIBRARY_GPR_PATH} ${GRPC_LIBRARY_GRPC_CPP_PATH}")
    else()
        message(FATAL_ERROR "Could not find gRPC library")
    endif()

    if (LINK_WITH_ABSL)
        target_link_libraries(gobgp_action absl::base absl::synchronization)
    endif()

    find_path(PROTOCOL_BUFFERS_INCLUDE_FOLDER NAMES "google/protobuf/stubs/common.h" PATHS "${PROTOCOL_BUFFERS_CUSTOM_INSTALL_PATH}/include")

    find_library(PROTOCOL_BUFFERS_LIBRARY_PATH NAMES protobuf PATHS "${PROTOCOL_BUFFERS_CUSTOM_INSTALL_PATH}/lib") 

    if (PROTOCOL_BUFFERS_INCLUDE_FOLDER AND PROTOCOL_BUFFERS_LIBRARY_PATH)
        include_directories(${PROTOCOL_BUFFERS_INCLUDE_FOLDER}) 
        target_link_libraries(gobgp_action ${PROTOCOL_BUFFERS_LIBRARY_PATH}) 
	    message(STATUS "Found protobuf library: ${PROTOCOL_BUFFERS_LIBRARY_PATH}")
    else()
        message(FATAL_ERROR "Could not find protocol buffers")
    endif()


    # message(STATUS "grpc: ${GRPC_INCLUDES_FOLDER} ${GRPC_LIBRARY_GRPC_PATH} ${GRPC_LIBRARY_GPR_PATH}")
    # message(STATUS ${PROJECT_BINARY_DIR})

    find_program(PROTOC_BINARY protoc PATHS "${PROTOCOL_BUFFERS_CUSTOM_INSTALL_PATH}/bin" ${DISABLE_DEFAULT_PATH_SEARCH_VAR})

    if (PROTOC_BINARY)
        message(STATUS "Found protoc protobuf compiler: ${PROTOC_BINARY}")
    else()
        message(FATAL_ERROR "Can't find protoc compiler")
    endif()

    # Search for gRPC plugin for protobuf, it's just binary
    find_program(GRPC_CPP_PLUGIN grpc_cpp_plugin PATHS "${GRPC_CUSTOM_INSTALL_PATH}/bin" ${DISABLE_DEFAULT_PATH_SEARCH_VAR})

    if (GRPC_CPP_PLUGIN) 
        message(STATUS "Found Protobuf gRPC compiler plugin: ${GRPC_CPP_PLUGIN}")
    else()
        message(FATAL_ERROR "Can't find Protobuf gRPC compiler plugin")
    endif()

    message(STATUS "Building protobuf and gRPC mappings for C++")

    execute_process(COMMAND ${PROTOC_BINARY} -I ${PROJECT_SOURCE_DIR}/actions --grpc_out=${PROJECT_SOURCE_DIR}/actions --plugin=protoc-gen-grpc=${GRPC_CPP_PLUGIN} ${PROJECT_SOURCE_DIR}/actions/gobgp.proto ERROR_VARIABLE PROTOC_STDERR RESULT_VARIABLE PROTOC_RETURN_CODE OUTPUT_STRIP_TRAILING_WHITESPACE)

    message(STATUS "Protoc return code for gobgp.proto gRPC: ${PROTOC_RETURN_CODE} std err: ${PROTOC_STDERR}")

    execute_process(COMMAND ${PROTOC_BINARY} -I  ${PROJECT_SOURCE_DIR}/actions --cpp_out=${PROJECT_SOURCE_DIR}/actions ${PROJECT_SOURCE_DIR}/actions/gobgp.proto ${PROJECT_SOURCE_DIR}/actions/attribute.proto  ERROR_VARIABLE PROTOC_STDERR RESULT_VARIABLE PROTOC_RETURN_CODE OUTPUT_STRIP_TRAILING_WHITESPACE)

    message(STATUS "Protoc return code for gobgp.proto and attribute.proto Protobuf: ${PROTOC_RETURN_CODE} std err: ${PROTOC_STDERR}")

    # Build gRPC and protocol bufffers libraries and link they to gobgp_action
    add_library(gobgp_api_client_pb_cc STATIC actions/gobgp.pb.cc)
    add_library(gobgp_api_client_grpc_pb_cc STATIC actions/gobgp.grpc.pb.cc)

    # Disable annoying warnings
    set_source_files_properties(actions/gobgp.pb.cc PROPERTIES COMPILE_FLAGS -Wno-deprecated-declarations)
    set_source_files_properties(actions/gobgp.grpc.pb.cc PROPERTIES COMPILE_FLAGS -Wno-deprecated-declarations)

    set_source_files_properties(actions/gobgp.pb.h PROPERTIES COMPILE_FLAGS -Wno-deprecated-declarations)
    set_source_files_properties(actions/gobgp.grpc.pb.h PROPERTIES COMPILE_FLAGS -Wno-deprecated-declarations)

    target_link_libraries(gobgp_action gobgp_api_client_pb_cc)
    target_link_libraries(gobgp_action gobgp_api_client_grpc_pb_cc)

    # Add attributes
    add_library(attribute_pb_cc STATIC actions/attribute.pb.cc)
    target_link_libraries(gobgp_action attribute_pb_cc)

    # FastNetMon API
    add_definitions(-DFASTNETMON_API)    

    execute_process(COMMAND ${PROTOC_BINARY} -I ${PROJECT_SOURCE_DIR} --grpc_out=${PROJECT_SOURCE_DIR} --plugin=protoc-gen-grpc=${GRPC_CPP_PLUGIN} ${PROJECT_SOURCE_DIR}/fastnetmon.proto ERROR_VARIABLE PROTOC_STDERR RESULT_VARIABLE PROTOC_RETURN_CODE OUTPUT_STRIP_TRAILING_WHITESPACE)

    message(STATUS "Protoc return code for gRPC fastnetmon.proto: ${PROTOC_RETURN_CODE} std err: ${PROTOC_STDERR}")

    execute_process(COMMAND ${PROTOC_BINARY} -I ${PROJECT_SOURCE_DIR} --cpp_out=${PROJECT_SOURCE_DIR} ${PROJECT_SOURCE_DIR}/fastnetmon.proto ERROR_VARIABLE PROTOC_STDERR RESULT_VARIABLE PROTOC_RETURN_CODE OUTPUT_STRIP_TRAILING_WHITESPACE) 

    message(STATUS "Protoc return code for protobuf fastnetmon.proto: ${PROTOC_RETURN_CODE} std err: ${PROTOC_STDERR}")

    # Disable annoying warnings
    set_source_files_properties(fastnetmon.grpc.pb.cc PROPERTIES COMPILE_FLAGS -Wno-deprecated-declarations)
    set_source_files_properties(fastnetmon.pb.cc PROPERTIES COMPILE_FLAGS -Wno-deprecated-declarations)
    set_source_files_properties(fastnetmon.grpc.pb.h PROPERTIES COMPILE_FLAGS -Wno-deprecated-declarations)
    set_source_files_properties(fastnetmon.pb.h PROPERTIES COMPILE_FLAGS -Wno-deprecated-declarations)

    add_library(fastnetmon_grpc_pb_cc STATIC fastnetmon.grpc.pb.cc)
    add_library(fastnetmon_pb_cc STATIC fastnetmon.pb.cc)

    add_executable(fastnetmon_api_client fastnetmon_api_client.cpp)

    if (LINK_WITH_ABSL)
        target_link_libraries(fastnetmon_api_client  absl::base absl::synchronization)
    endif()

    target_link_libraries(fastnetmon_api_client ${GRPC_LIBRARY_GPR_PATH})
    target_link_libraries(fastnetmon_api_client ${GRPC_LIBRARY_GRPC_CPP_PATH})
    target_link_libraries(fastnetmon_api_client ${GRPC_LIBRARY_GRPC_PATH})
    target_link_libraries(fastnetmon_api_client fastnetmon_grpc_pb_cc)
    target_link_libraries(fastnetmon_api_client fastnetmon_pb_cc)
    target_link_libraries(fastnetmon_api_client ${PROTOCOL_BUFFERS_LIBRARY_PATH}) 

    target_link_libraries(fastnetmon ${GRPC_LIBRARY_GPR_PATH})
    target_link_libraries(fastnetmon ${GRPC_LIBRARY_GRPC_CPP_PATH})
    target_link_libraries(fastnetmon ${GRPC_LIBRARY_GRPC_PATH})
    target_link_libraries(fastnetmon fastnetmon_grpc_pb_cc)
    target_link_libraries(fastnetmon fastnetmon_pb_cc)
    target_link_libraries(fastnetmon ${PROTOCOL_BUFFERS_LIBRARY_PATH}) 
endif()

if (ENABLE_CUSTOM_BOOST_BUILD)
    target_link_libraries(fastnetmon ${ICU_LIBRARY_UC_PATH})
    target_link_libraries(fastnetmon ${ICU_LIBRARY_DATA_PATH})
    target_link_libraries(fastnetmon ${ICU_LIBRARY_I18N_PATH})
endif()

# example plugin
add_library(example_plugin STATIC example_plugin/example_collector.cpp)

if (ENABLE_NETMAP_SUPPORT)
    # Netmap plugin
    set(NETMAP_INCLUDE_DIRS "netmap_plugin/netmap_includes")
    include_directories(${NETMAP_INCLUDE_DIRS})
    add_library(netmap_plugin STATIC netmap_plugin/netmap_collector.cpp)
endif()

# Client tool
add_executable(fastnetmon_client fastnetmon_client.cpp)

# Find boost: http://www.cmake.org/cmake/help/v3.0/module/FindBoost.html

# Enable detailed errors
set(Boost_DETAILED_FAILURE_MSG ON)
# set(Boost_DEBUG ON)

find_package(Boost COMPONENTS thread regex program_options system REQUIRED)

if(Boost_FOUND)
    message(STATUS "Found Boost: ${Boost_LIBRARIES} ${Boost_INCLUDE_DIRS}")

    include_directories(${Boost_INCLUDE_DIRS})
    target_link_libraries(fastnetmon ${Boost_LIBRARIES})
    target_link_libraries(fast_library ${Boost_LIBRARIES})
    target_link_libraries(fastnetmon_client ${Boost_LIBRARIES})
endif()

target_link_libraries(fast_library patricia)
target_link_libraries(fast_library fastnetmon_pcap_format)

# Try to find ncurses library
find_package(Curses REQUIRED)

if(CURSES_FOUND) 
    include_directories(${CURSES_INCLUDE_DIRS})
    target_link_libraries(fastnetmon_client ${CURSES_LIBRARIES})
endif()

### Move this code to cmake module

# Try to find hiredis in a specific folder
find_path(HIREDIS_INCLUDES_FOLDER NAMES hiredis/hiredis.h PATHS "${HIREDIS_CUSTOM_INSTALL_PATH}/include" ${DISABLE_DEFAULT_PATH_SEARCH_VAR})

# Try to find hiredis library path
find_library(HIREDIS_LIBRARY_PATH NAMES hiredis PATHS "${HIREDIS_CUSTOM_INSTALL_PATH}/lib" ${DISABLE_DEFAULT_PATH_SEARCH_VAR})

if (HIREDIS_INCLUDES_FOLDER AND HIREDIS_LIBRARY_PATH) 
    message(STATUS "We found hiredis library ${HIREDIS_INCLUDES_FOLDER} ${HIREDIS_LIBRARY_PATH}")

    add_definitions(-DREDIS)
    include_directories(${HIREDIS_INCLUDES_FOLDER})
    target_link_libraries (fastnetmon ${HIREDIS_LIBRARY_PATH})
    target_link_libraries(fastnetmon_logic ${HIREDIS_LIBRARY_PATH})
else()
    message(STATUS "We can't find hiredis library and will disable Redis support")
endif()

set(ENABLE_OPENSSL_SUPPORT TRUE)
if (ENABLE_OPENSSL_SUPPORT)
    find_path(OPENSSL_INCLUDES_FOLDER NAMES "openssl/rsa.h" PATHS "${OPENSSL_CUSTOM_INSTALL_PATH}/include" ${DISABLE_DEFAULT_PATH_SEARCH_VAR})
    
    # Check that we found headers
    if (OPENSSL_INCLUDES_FOLDER) 
        message(STATUS "We found OpenSSL library headers: ${OPENSSL_INCLUDES_FOLDER}")
        include_directories(${OPENSSL_INCLUDES_FOLDER})
    else()
        message(FATAL_ERROR "Could not find OpenSSL headers")
    endif()

    find_library(OPENSSL_LIBRARY_PATH NAMES ssl PATHS "${OPENSSL_CUSTOM_INSTALL_PATH}/lib" ${DISABLE_DEFAULT_PATH_SEARCH_VAR})
    find_library(OPENSSL_CRYPTO_LIBRARY_PATH NAMES crypto PATHS "${OPENSSL_CUSTOM_INSTALL_PATH}/lib" ${DISABLE_DEFAULT_PATH_SEARCH_VAR})

    # Check that we found libraries
    if (OPENSSL_LIBRARY_PATH AND OPENSSL_CRYPTO_LIBRARY_PATH)
        message(STATUS "We found OpenSSL library: ${OPENSSL_LIBRARY_PATH} ${OPENSSL_CRYPTO_LIBRARY_PATH}")
    else()
        message(FATAL_ERROR "Could not find OpenSSL libraries")
    endif()
endif()

if (ENABLE_CAPNP_SUPPORT)
    add_definitions(-DENABLE_CAPNP)

    find_library(CAPNP_LIBRARY_PATH NAMES capnp PATHS "${CAPNP_CUSTOM_INSTALL_PATH}/lib" ${DISABLE_DEFAULT_PATH_SEARCH_VAR})
    find_library(CAPNP_KJ_LIBRARY_PATH NAMES kj PATHS "${CAPNP_CUSTOM_INSTALL_PATH}/lib" ${DISABLE_DEFAULT_PATH_SEARCH_VAR})

    if (CAPNP_LIBRARY_PATH AND CAPNP_KJ_LIBRARY_PATH)
        message(STATUS "We found capnp and kj libraries: ${CAPNP_LIBRARY_PATH} ${CAPNP_KJ_LIBRARY_PATH}")
    else()
        message(FATAL_ERROR "Could not find capnp libraries")
    endif()

    include_directories("${CAPNP_CUSTOM_INSTALL_PATH}/include")

    target_link_libraries(simple_packet_capnp ${CAPNP_LIBRARY_PATH} ${CAPNP_KJ_LIBRARY_PATH})

    # Link it with cap'n'p stuff
    target_link_libraries(fast_library simple_packet_capnp)

endif()


### Find mongo-c
find_path(MONGOC_INCLUDES_FOLDER NAMES libmongoc-1.0/mongoc.h PATHS "${MONGO_C_CUSTOM_INSTALL_PATH}/include" ${DISABLE_DEFAULT_PATH_SEARCH_VAR})
find_library(MONGOC_LIBRARY_PATH NAMES mongoc-1.0 PATHS "${MONGO_C_CUSTOM_INSTALL_PATH}/lib" ${DISABLE_DEFAULT_PATH_SEARCH_VAR})

### find bson
find_path(BSON_INCLUDES_FOLDER NAMES libbson-1.0/bson.h PATHS "${MONGO_C_CUSTOM_INSTALL_PATH}/include" ${DISABLE_DEFAULT_PATH_SEARCH_VAR})
find_library(BSON_LIBRARY_PATH NAMES bson-1.0 PATHS "${MONGO_C_CUSTOM_INSTALL_PATH}/lib" ${DISABLE_DEFAULT_PATH_SEARCH_VAR})

if (MONGOC_INCLUDES_FOLDER AND MONGOC_LIBRARY_PATH AND BSON_INCLUDES_FOLDER AND BSON_LIBRARY_PATH)
    message(STATUS "We found mongo-c library ${MONGOC_INCLUDES_FOLDER} ${MONGOC_LIBRARY_PATH} ${BSON_INCLUDES_FOLDER} ${BSON_LIBRARY_PATH}")
    add_definitions(-DMONGO)

    # We add suffix name because cmake could not detect it correctly...
    include_directories("${MONGOC_INCLUDES_FOLDER}/libmongoc-1.0")
    include_directories("${BSON_INCLUDES_FOLDER}/libbson-1.0")

    target_link_libraries(fastnetmon ${MONGOC_LIBRARY_PATH} ${BSON_LIBRARY_PATH})
    target_link_libraries(fastnetmon_logic ${MONGOC_LIBRARY_PATH} ${BSON_LIBRARY_PATH})
else()
    message(FATAL_ERROR "We can't find Mongo C library")
endif()

### Look for libpcap

#find_path(LIBPCAP_INCLUDES_FOLDER NAMES pcap.h PATHS "${LIBPCAP_CUSTOM_INSTALL_PATH}/include" ${DISABLE_DEFAULT_PATH_SEARCH_VAR})

#find_library(LIBPCAP_LIBRARY_PATH NAMES pcap PATHS "${LIBPCAP_CUSTOM_INSTALL_PATH}/lib" ${DISABLE_DEFAULT_PATH_SEARCH_VAR})

#if (LIBPCAP_INCLUDES_FOLDER AND LIBPCAP_LIBRARY_PATH) 
#    message(STATUS "We found pcap library ${LIBPCAP_LIBRARY_PATH}")

#    include_directories(${LIBPCAP_INCLUDES_FOLDER})
#else()
#    message(FATAL_ERROR "We can't find pcap library")    
#endif()

### Look for log4cpp

# Try to find log4cpp includes path
find_path(LOG4CPP_INCLUDES_FOLDER NAMES log4cpp/Appender.hh PATHS "${LOG4CPP_CUSTOM_INSTALL_PATH}/include" ${DISABLE_DEFAULT_PATH_SEARCH_VAR})

# Try to find log4cpp library path
find_library(LOG4CPP_LIBRARY_PATH NAMES log4cpp PATHS "${LOG4CPP_CUSTOM_INSTALL_PATH}/lib" ${DISABLE_DEFAULT_PATH_SEARCH_VAR})

if (LOG4CPP_INCLUDES_FOLDER AND LOG4CPP_LIBRARY_PATH)
    include_directories(${LOG4CPP_INCLUDES_FOLDER})
    message(STATUS "We have found log4cpp: ${LOG4CPP_LIBRARY_PATH}")
else()
    message(FATAL_ERROR "We can't find log4cpp. We can't build project")
endif()

### Look for jsonc

target_link_libraries(fast_library ${JSONC_LIBRARY_PATH})

target_link_libraries(fast_library ${OPENSSL_LIBRARY_PATH})
target_link_libraries(fast_library ${OPENSSL_CRYPTO_LIBRARY_PATH})

target_link_libraries(fastnetmon ${LOG4CPP_LIBRARY_PATH})

target_link_libraries(fastnetmon ${CMAKE_THREAD_LIBS_INIT})

# Our libs
target_link_libraries(fastnetmon patricia)
target_link_libraries(fastnetmon fastnetmon_pcap_format)

target_link_libraries(fastnetmon ipfix_rfc)

target_link_libraries(fastnetmon_logic bgp_protocol exabgp_action)

# Link to our functions
target_link_libraries(fastnetmon fast_library)

# link to our unified parser
target_link_libraries(fastnetmon ${OPENSSL_LIBRARY_PATH})
target_link_libraries(fastnetmon ${OPENSSL_CRYPTO_LIBRARY_PATH})

if (ENABLE_GOBGP_SUPPORT)
    target_link_libraries(fastnetmon gobgp_action)
endif()

target_link_libraries(fastnetmon exabgp_action)

if (ENABLE_AFPACKET_SUPPORT)
    target_link_libraries(fastnetmon afpacket_plugin)
endif()

target_link_libraries(fastnetmon sflow_plugin netflow_plugin pcap_plugin example_plugin)

target_link_libraries(fastnetmon fastnetmon_logic)

if (ENABLE_NETMAP_SUPPORT)
    target_link_libraries(fastnetmon netmap_plugin)
endif()

# According to YunQiang Su debian-mips@lists.debian.org 
# Due to the limitation of gnu ld, -latomic should be put after library which calls it
# I decided that keeping it here as very last dependency is pretty good option to guarantee it
if (LINK_WITH_ATOMIC_LIBRARY) 
    target_link_libraries(fastnetmon atomic)
endif()

# cmake .. -DBUILD_PLUGIN_RUNNER=ON
if (BUILD_PLUGIN_RUNNER)
    add_executable(fastnetmon_plugin_runner plugin_runner.cpp)

    if (ENABLE_AFPACKET_SUPPORT) 
        target_link_libraries(fastnetmon_plugin_runner afpacket_plugin)
    endif()

    target_link_libraries(fastnetmon_plugin_runner ${CMAKE_THREAD_LIBS_INIT})
    target_link_libraries(fastnetmon_plugin_runner patricia)
    target_link_libraries(fastnetmon_plugin_runner fastnetmon_pcap_format)
    target_link_libraries(fastnetmon_plugin_runner ${LOG4CPP_LIBRARY_PATH})
    target_link_libraries(fastnetmon_plugin_runner fast_library)

    # Add all plugins
    target_link_libraries(fastnetmon_plugin_runner sflow_plugin netflow_plugin pcap_plugin example_plugin)

    if (ENABLE_NETMAP_SUPPORT)
        target_link_libraries(fastnetmon_plugin_runner netmap_plugin)
    endif()
endif()

# cmake .. -DBUILD_PCAP_READER=ON
if (BUILD_PCAP_READER)
    add_executable(fastnetmon_pcap_reader pcap_reader.cpp)

    target_link_libraries(fastnetmon_pcap_reader patricia)
    target_link_libraries(fastnetmon_pcap_reader fastnetmon_pcap_format)

    target_link_libraries(fastnetmon_pcap_reader fast_library)
    target_link_libraries(fastnetmon_pcap_reader ${LOG4CPP_LIBRARY_PATH})
    target_link_libraries(fastnetmon_pcap_reader netflow_plugin)   
    target_link_libraries(fastnetmon_pcap_reader sflow_plugin)

    if (ENABLE_NETMAP_SUPPORT)
        target_link_libraries(fastnetmon_pcap_reader netmap_plugin)
    endif()
endif()

# cmake -DBUILD_TESTS=ON ..
if (BUILD_TESTS) 
    add_executable(fastnetmon_tests fastnetmon_tests.cpp)
    target_link_libraries(fastnetmon_tests fast_library)
    target_link_libraries(fastnetmon_tests ${CMAKE_THREAD_LIBS_INIT})
    target_link_libraries(fastnetmon_tests ${Boost_LIBRARIES})
    target_link_libraries(fastnetmon_tests ${LOG4CPP_LIBRARY_PATH})

    set(GOOGLE_TEST_INCLUDE_DIRS ${FASTNETMON_LIBRARIES_GLOBAL_PATH}/gtest/include)
    set(GOOGLE_TEST_LIBRARIES ${FASTNETMON_LIBRARIES_GLOBAL_PATH}/gtest/lib/libgtest.a ${FASTNETMON_LIBRARIES_GLOBAL_PATH}/gtest/lib/libgtest_main.a)

    # Compiled Google Library
    include_directories(${GOOGLE_TEST_INCLUDE_DIRS})
    target_link_libraries(fastnetmon_tests ${GOOGLE_TEST_LIBRARIES})

    add_executable(traffic_structures_tests tests/traffic_structures_performance_tests.cpp)
    target_link_libraries(traffic_structures_tests ${Boost_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT} ${LOG4CPP_LIBRARY_PATH} fast_library)

    add_executable(traffic_structures_tests_real_traffic tests/traffic_structures_performance_tests_real_traffic.cpp)
    target_link_libraries(traffic_structures_tests_real_traffic ${Boost_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT} ${LOG4CPP_LIBRARY_PATH} fast_library)

    add_executable(patricia_performance_tests tests/patricia_performance_tests.cpp)
    target_link_libraries(patricia_performance_tests patricia fast_library ${LOG4CPP_LIBRARY_PATH})
endif()

# Check default values prepared by CMAKE for us
message(STATUS "Install BINDIR path: ${CMAKE_INSTALL_BINDIR}")
message(STATUS "Install SBINDIR path: ${CMAKE_INSTALL_SBINDIR}")
message(STATUS "Install SYSCONFDIR path: ${CMAKE_INSTALL_SYSCONFDIR}")
message(STATUS "Install MANDIR path: ${CMAKE_INSTALL_MANDIR}")

# We use this flag on Debian upstream builds because we apparently need absolute paths here
# But for Homebrew we need option to disable it and use relative paths
# cmake .. -DSET_ABSOLUTE_INSTALL_PATH=OFF
option(SET_ABSOLUTE_INSTALL_PATH "Enables use of absolute install paths" ON)

if (${CMAKE_SYSTEM_NAME} STREQUAL "FreeBSD" OR ${CMAKE_SYSTEM_NAME} STREQUAL "DragonFly")
    set(CMAKE_INSTALL_BINDIR "bin")
    set(CMAKE_INSTALL_SBINDIR "bin")
    set(CMAKE_INSTALL_SYSCONFDIR "etc")
elseif (${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
    if (SET_ABSOLUTE_INSTALL_PATH)
        set(CMAKE_INSTALL_BINDIR "/usr/bin")
        set(CMAKE_INSTALL_SBINDIR "/usr/sbin")
        set(CMAKE_INSTALL_SYSCONFDIR "/etc")
        set(CMAKE_INSTALL_MANDIR "/usr/share/man")
    endif()
elseif (${CMAKE_SYSTEM_NAME} STREQUAL "Darwin")
    message(STATUS "We run on Apple platform")
else()
    message(STATUS "We run on platform ${CMAKE_SYSTEM_NAME} and we do not touch install paths")
    # Do not touch these variables and use default values
endif()

install(TARGETS fastnetmon DESTINATION "${CMAKE_INSTALL_SBINDIR}")
install(TARGETS fastnetmon_client DESTINATION "${CMAKE_INSTALL_BINDIR}")
install(TARGETS fastnetmon_api_client DESTINATION "${CMAKE_INSTALL_BINDIR}")

install(FILES fastnetmon.conf DESTINATION "${CMAKE_INSTALL_SYSCONFDIR}")

# Install blank files for networks list and whitelist
install(FILES networks_list DESTINATION "${CMAKE_INSTALL_SYSCONFDIR}")
install(FILES networks_whitelist DESTINATION "${CMAKE_INSTALL_SYSCONFDIR}")

# man pages
install(FILES man/fastnetmon.8 DESTINATION ${CMAKE_INSTALL_MANDIR}/man8)
install(FILES man/fastnetmon_client.1 DESTINATION ${CMAKE_INSTALL_MANDIR}/man1)

if (SET_ABSOLUTE_INSTALL_PATH)
    # Unfortunately, we have no cross-platform option to install systemd units in current versions of cmake
    set(CMAKE_INSTALL_SYSTEMD_SERVICEDIR "/lib/systemd/system"  CACHE PATH "Location for systemd service files")

    # Generate unit file
    if(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
        configure_file(fastnetmon.service.in "${CMAKE_CURRENT_BINARY_DIR}/fastnetmon.service" @ONLY)
        install(FILES "${CMAKE_CURRENT_BINARY_DIR}/fastnetmon.service" DESTINATION ${CMAKE_INSTALL_SYSTEMD_SERVICEDIR})
    endif()
else()
    # We have no relative standard path for installation, skip any actions
endif()

# For Debian packages build please use our logic from upstream: https://salsa.debian.org/debian/fastnetmon

# Configure cpack package builder
# Run it with: cd build; cpack -G DEB ..
set(CPACK_PACKAGE_NAME "fastnetmon")
set(CPACK_PACKAGE_VENDOR "fastnetmon.com")
set(CPACK_PACKAGE_CONTACT "pavel.odintsov@gmail.com")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "FastNetMon - very fast DoS/DDoS detector with sFlow/Netflow/mirror support")
set(CPACK_DEBIAN_PACKAGE_DEPENDS "")
# set(CPACK_PACKAGE_INSTALL_DIRECTORY "CPack Component Example")

if (NOT DO_NOT_USE_SYSTEM_LIBRARIES_FOR_BUILD)

# Specify config for deb package
# http://www.cmake.org/Wiki/CMake:CPackPackageGenerators#DEB_.28UNIX_only.29
# These dependencies are for Debian Bullseye
set(CPACK_DEBIAN_PACKAGE_DEPENDS "libboost-thread-dev, libboost-system-dev, libboost-regex-dev, libpcap-dev, libnuma-dev, liblog4cpp5-dev, libgrpc10, libgrpc++1, libcapnp-0.7.0, libmongoc-1.0-0, libbson-1.0-0, libboost-program-options1.74.0")

endif()

# This must always be last!
include(CPack)

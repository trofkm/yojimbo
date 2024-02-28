#
# Author: Kirill Trofimov
# Email: trofkm@gmail.com
# Date: 28.02.24
#

cmake_minimum_required(VERSION 3.10)

find_program(CCACHE_FOUND ccache)
if (CCACHE_FOUND)
    set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ccache)
    set(CMAKE_C_COMPILER_LAUNCHER "ccache")
endif (CCACHE_FOUND)


set(CMAKE_CXX_STANDARD 14)
set(CMAKE_C_STANDARD 99)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_INCLUDE_CURRENT_DIR ON)

option(BUILD_SHARED "Build yojimbo as shared library" OFF)
option(BUILD_EXAMPLES "Build examples" ON)
option(BUILD_TESTS "Build tests" ON)


project(yojimbo LANGUAGES C CXX)


# deps


file(GLOB reliable_sources "libs/reliable/*")
add_library(reliable STATIC ${reliable_sources})
set_target_properties(reliable PROPERTIES LINKER_LANGUAGE C)
target_include_directories(reliable PUBLIC libs/reliable)

file(GLOB serialize_sources "libs/serialize/*")
add_library(serialize STATIC ${serialize_sources})
set_target_properties(serialize PROPERTIES LINKER_LANGUAGE C)
target_include_directories(serialize PUBLIC libs/serialize)


file(GLOB sodium_sources "thirdparty/sodium/*")
add_library(sodium STATIC ${sodium_sources})
set_target_properties(sodium PROPERTIES LINKER_LANGUAGE C)
target_include_directories(sodium PUBLIC thirdparty/sodium)

file(GLOB tlsf_sources "thirdparty/tlsf/*")
add_library(tlsf STATIC ${tlsf_sources})
# TODO: According to initial premake it is okay, but I do not like such directory include
target_include_directories(tlsf PUBLIC thirdparty)
set_target_properties(tlsf PROPERTIES LINKER_LANGUAGE C)

file(GLOB netcode_sources "libs/netcode/*")
add_library(netcode STATIC ${netcode_sources})
set_target_properties(netcode PROPERTIES LINKER_LANGUAGE C)
target_include_directories(netcode PUBLIC libs/netcode)
target_link_libraries(netcode PRIVATE sodium)


# yojimbo
file(GLOB yojimbo_sources "${CMAKE_CURRENT_SOURCE_DIR}/source/*")
if (BUILD_SHARED)
    add_library(yojimbo SHARED ${yojimbo_sources})
else ()
    add_library(yojimbo STATIC ${yojimbo_sources})
endif ()

set_target_properties(yojimbo PROPERTIES LINKER_LANGUAGE CXX)
target_include_directories(yojimbo PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
target_link_libraries(yojimbo PUBLIC tlsf serialize reliable netcode)

# examples

if (BUILD_EXAMPLES)

    # client-server communication

    add_executable(client examples/client.cpp)
    target_link_libraries(client yojimbo)

    add_executable(server examples/server.cpp)
    target_link_libraries(server yojimbo)

    # loopback

    add_executable(loopback examples/loopback.cpp)
    target_link_libraries(loopback yojimbo)

    # soak

    add_executable(soak examples/soak.cpp)
    target_link_libraries(soak yojimbo)
endif ()

# tests

if(BUILD_TESTS)
    enable_testing()
    target_compile_definitions(serialize PUBLIC -DSERIALIZE_ENABLE_TESTS)
    target_compile_definitions(netcode PUBLIC -DNETCODE_ENABLE_TESTS)
    target_compile_definitions(reliable PUBLIC -DRELIABLE_ENABLE_TESTS)

    add_executable(test_all tests/test.cpp)
    target_link_libraries(test_all yojimbo)

    add_test(NAME run_all COMMAND test_main)
endif ()

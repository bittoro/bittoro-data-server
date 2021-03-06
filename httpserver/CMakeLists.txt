cmake_minimum_required(VERSION 3.1)

add_definitions(-DDISABLE_ENCRYPTION)

project(httpserver)

set(HEADER_FILES
    http_connection.h
    swarm.h
    service_node.h
    serialization.h
    rate_limiter.h
    ../external/json.hpp
    https_client.h
    server_certificates.h
    stats.h
    security.h
    command_line.h
    net_stats.h
    dns_text_records.h
    reachability_testing.h
    )

set(SRC_FILES
    main.cpp
    http_connection.cpp
    swarm.cpp
    service_node.cpp
    serialization.cpp
    rate_limiter.cpp
    https_client.cpp
    stats.cpp
    security.cpp
    command_line.cpp
    dns_text_records.cpp
    reachability_testing.cpp
    )

add_library(httpserver_lib STATIC ${HEADER_FILES} ${SRC_FILES})

loki_add_subdirectory(../common common)
loki_add_subdirectory(../storage storage)
loki_add_subdirectory(../utils utils)
loki_add_subdirectory(../pow pow)
loki_add_subdirectory(../crypto crypto)
find_package(OpenSSL REQUIRED)

find_package(Boost
    REQUIRED
    thread
    system
    filesystem
    program_options
)

target_link_libraries(httpserver_lib PUBLIC common)
target_link_libraries(httpserver_lib PUBLIC OpenSSL::SSL OpenSSL::Crypto)
target_link_libraries(httpserver_lib PUBLIC storage)
target_link_libraries(httpserver_lib PUBLIC utils)
target_link_libraries(httpserver_lib PUBLIC pow)
target_link_libraries(httpserver_lib PUBLIC resolv)
target_link_libraries(httpserver_lib PUBLIC crypto)

set_property(TARGET httpserver_lib PROPERTY CXX_STANDARD 14)

target_include_directories(httpserver_lib PUBLIC ${CMAKE_CURRENT_SOURCE_DIR} ${Boost_INCLUDE_DIRS})
target_link_libraries(httpserver_lib PUBLIC ${Boost_LIBRARIES})

set(BIN_NAME data-server)

add_executable(httpserver main.cpp)
set_target_properties(httpserver PROPERTIES OUTPUT_NAME ${BIN_NAME})
set_property(TARGET httpserver PROPERTY CXX_STANDARD 14)
target_link_libraries(httpserver PRIVATE httpserver_lib)
install(TARGETS httpserver DESTINATION bin)
# Build Info
execute_process(
    COMMAND
        git rev-parse --short HEAD
    OUTPUT_VARIABLE
        SHORT_HASH
    OUTPUT_STRIP_TRAILING_WHITESPACE)
string(TIMESTAMP BUILD_TIME UTC)
message(STATUS "using git commit hash ${SHORT_HASH}")
message(STATUS "using UTC build time ${BUILD_TIME}")
target_compile_definitions(httpserver PRIVATE -DSTORAGE_SERVER_GIT_HASH_STRING="${SHORT_HASH}")
target_compile_definitions(httpserver PRIVATE -DSTORAGE_SERVER_BUILD_TIME="${BUILD_TIME}")

find_package(PkgConfig QUIET)
if(PKG_CONFIG_FOUND)
    pkg_check_modules(SYSTEMD libsystemd)
    # Default ENABLE_SYSTEMD to true if we found it
    option(ENABLE_SYSTEMD "enable systemd integration for sd_notify" ${SYSTEMD_FOUND})

    if(ENABLE_SYSTEMD)
        if(NOT SYSTEMD_FOUND)
            message(FATAL_ERROR "libsystemd not found")
        endif()
        target_compile_definitions(httpserver PRIVATE ENABLE_SYSTEMD)
        target_include_directories(httpserver PRIVATE ${SYSTEMD_INCLUDE_DIRS})
        target_link_libraries(httpserver PRIVATE ${SYSTEMD_LIBRARIES})
    endif()
endif()


#

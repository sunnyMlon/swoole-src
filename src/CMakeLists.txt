PROJECT(libswoole)
CMAKE_MINIMUM_REQUIRED(VERSION 2.8)

SET(SWOOLE_VERSION 4.5.3-alpha)

enable_language(ASM)

set(CMAKE_CXX_STANDARD 11)

add_definitions(-DHAVE_CONFIG_H)

set(libswoole_libraries pthread dl)
set(libswoole_headers_dir ${CMAKE_CURRENT_SOURCE_DIR}/../include)
set(libswoole_includes ${CMAKE_CURRENT_SOURCE_DIR}/.. ${libswoole_headers_dir} ${libswoole_headers_dir}/wrapper)
set(libswoole_thirdparty_dir ${CMAKE_CURRENT_SOURCE_DIR}/../thirdparty)

if (APPLE)
    set(CMAKE_SHARED_LINKER_FLAGS "-undefined dynamic_lookup")
else()
    list(APPEND libswoole_libraries rt crypt)
endif()

SET(CMAKE_BUILD_TYPE Debug)

# Code Coverage Configuration
add_library(coverage_config INTERFACE)

option(CODE_COVERAGE "Enable coverage reporting" OFF)
if(CODE_COVERAGE)
    message(STATUS "Open coverage")
    # --coverage => -fprofile-arcs -ftest-coverage
    target_compile_options(coverage_config INTERFACE
        -O0
        -g
        --coverage
    )
    if(CMAKE_VERSION VERSION_GREATER_EQUAL 3.13)
        target_link_options(coverage_config INTERFACE --coverage)
    else()
        target_link_libraries(coverage_config INTERFACE --coverage)
    endif()
endif(CODE_COVERAGE)

file(GLOB libswoole_sources_core core/*.cc)
file(GLOB libswoole_sources_coroutine coroutine/*.cc)
file(GLOB libswoole_sources_lock lock/*.cc)
file(GLOB libswoole_sources_memory memory/*.cc)
file(GLOB libswoole_sources_network network/*.cc)
file(GLOB libswoole_sources_os os/*.cc)
file(GLOB libswoole_sources_pipe pipe/*.cc)
file(GLOB libswoole_sources_protocol protocol/*.cc)
file(GLOB libswoole_sources_reactor reactor/*.cc)
file(GLOB libswoole_sources_server server/*.cc)
file(GLOB libswoole_sources_wrapper server/*.cc)
set(libswoole_sources_thirdparty
    ${libswoole_thirdparty_dir}/boost/asm/combined.S
)

list(APPEND libswoole_sources
    ${libswoole_sources_core}
    ${libswoole_sources_coroutine}
    ${libswoole_sources_lock}
    ${libswoole_sources_memory}
    ${libswoole_sources_network}
    ${libswoole_sources_os}
    ${libswoole_sources_pipe}
    ${libswoole_sources_protocol}
    ${libswoole_sources_reactor}
    ${libswoole_sources_server}
    ${libswoole_sources_wrapper}
    ${libswoole_sources_thirdparty}
)

file(GLOB_RECURSE HEAD_FILES FOLLOW_SYMLINKS ${libswoole_headers_dir}/*.h)
file(GLOB_RECURSE HEAD_WAPPER_FILES FOLLOW_SYMLINKS ${libswoole_headers_dir}/wrapper/*.hpp)

SET(LIBRARY_OUTPUT_PATH ${CMAKE_CURRENT_SOURCE_DIR}/lib)

# find OpenSSL
if (DEFINED openssl_dir) 
    include_directories(BEFORE ${openssl_dir}/include)
    list(APPEND libswoole_includes ${openssl_dir}/include)
    link_directories(${openssl_dir}/lib)
else()
    find_package(OpenSSL)
    if (${OPENSSL_FOUND})
        message(STATUS "Found OpenSSL")
        list(APPEND libswoole_includes ${OPENSSL_INCLUDE_DIR})
        list(APPEND libswoole_libraries ${OPENSSL_LIBRARIES})
    else()
        message(STATUS "Not found OpenSSL")
    endif()
endif()

if (DEFINED brotli_dir) 
    list(APPEND libswoole_includes ${brotli_dir}/include)
    link_directories(${brotli_dir}/lib)
endif()

# lib-swoole
link_directories(${LIBRARY_OUTPUT_PATH})
add_library(lib-swoole SHARED ${libswoole_sources})
set_target_properties(lib-swoole PROPERTIES OUTPUT_NAME "swoole" VERSION ${SWOOLE_VERSION})
target_compile_options(lib-swoole PRIVATE ${CMAKE_CXX_FLAGS} -Wall -g)
target_include_directories(lib-swoole PRIVATE ${libswoole_includes})
target_link_libraries(lib-swoole ${libswoole_libraries})
if(CODE_COVERAGE)
    target_link_libraries(lib-swoole coverage_config gcov)
endif(CODE_COVERAGE)

#install
INSTALL(CODE "MESSAGE(\"Are you run command using root user?\")")
INSTALL(TARGETS lib-swoole LIBRARY DESTINATION lib ARCHIVE DESTINATION lib)
INSTALL(FILES ${HEAD_FILES} DESTINATION include/swoole)
INSTALL(FILES ${HEAD_WAPPER_FILES} DESTINATION include/swoole/wrapper)

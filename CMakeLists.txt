cmake_minimum_required (VERSION 2.8.11)
project(mproxy)

set(MPROXY_MAJOR_VERSION 0)
set(MPROXY_MINOR_VERSION 3)
set(MPROXY_PATCH_VERSION 0)

set(CMAKE_VERBOSE_MAKEFILE on)

#set(CMAKE_C_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -Wall -g -O0")
#set(CMAKE_C_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -Wall -O2")
if( NOT CMAKE_BUILD_TYPE )
    set( CMAKE_BUILD_TYPE Debug CACHE STRING
        "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel."
        FORCE )
endif()

# -DENABLE_SS:STRING=ON
OPTION(ENABLE_SS "Enable shadowsocks protocol" ON)
# -DUSE_CRYPTO_OPENSSL:STRING=ON
OPTION(USE_CRYPTO_OPENSSL "Use OpenSSL crypt library" ON)
OPTION(USE_CRYPTO_MBEDTLS "Use mbed TLS crypt library" OFF)
# -DENABLE_HTTPS_PROXY:STRING=ON
OPTION(ENABLE_HTTPS_PROXY "Enable https proxy" OFF)

# add source files to DIR_SRCS
aux_source_directory(. DIR_SRCS)

# libevhtp subdir
# add a variable of the same name to the cache, so options in subdirectory can be changed
if (NOT ENABLE_HTTPS_PROXY)
    set(EVHTP_DISABLE_SSL ON CACHE BOOL "Disable ssl")
endif()
set(EVHTP_DISABLE_EVTHR ON CACHE BOOL "Disable thread")
set(EVHTP_DISABLE_REGEX ON CACHE BOOL "Disable regex")
add_subdirectory(libevhtp)

include_directories(SYSTEM ${LIBEVENT_INCLUDE_DIR})
set(MPROXY_LIBS ${MPROXY_LIBS} ${LIBEVENT_LIBRARY})
if (WIN32)
    find_library(LIB_WS32 ws2_32)
    set(MPROXY_LIBS ${MPROXY_LIBS} ${LIB_WS32})
elseif (NOT APPLE)
    find_library(LIB_RT rt)
    set(MPROXY_LIBS ${MPROXY_LIBS} ${LIB_RT})
endif()

CHECK_FUNCTION_EXISTS(strerror_r HAVE_STRERROR_R)
if (HAVE_STRERROR_R)
    add_definitions(-DHAVE_STRERROR_R=1)
endif()

CHECK_FUNCTION_EXISTS(splice HAVE_SPLICE)
if (HAVE_SPLICE)
    add_definitions(-DHAVE_SPLICE=1)
endif()

set(MPROXY_SOURCES connector.c evhtp_proxy.c evhtp_sock_relay.c lru.c utils.c log.c )
if (MSVC)
    add_definitions(-D_WINSOCK_DEPRECATED_NO_WARNINGS -D_CRT_SECURE_NO_WARNINGS)
    set(MPROXY_SOURCES ${MPROXY_SOURCES} bsd_getopt.c )
endif()

if (USE_CRYPTO_OPENSSL)
    find_package(OpenSSL 1.1.0)
    if (OPENSSL_FOUND)
        include_directories(SYSTEM ${OPENSSL_INCLUDE_DIR})
        add_definitions(-DUSE_CRYPTO_OPENSSL=1)
        set (MPROXY_LIBS ${MPROXY_LIBS} ${OPENSSL_CRYPTO_LIBRARY})
    else()
        message(WARNING "Unable to find OpenSSL ")
        set(USE_CRYPTO_OPENSSL OFF)
    endif()
elseif (USE_CRYPTO_MBEDTLS)
    find_path(MBEDTLS_INCLUDE_DIR mbedtls/cipher.h 
        HINTS ${MBEDTLS_ROOT_DIR} 
        PATH_SUFFIXES include)
    find_library(MBEDCRYPTO_LIBRARY mbedcrypto 
        HINTS ${MBEDTLS_ROOT_DIR} 
        PATH_SUFFIXES library lib)
    if (MBEDTLS_INCLUDE_DIR AND MBEDCRYPTO_LIBRARY)
        include_directories(${MBEDTLS_INCLUDE_DIR})
        add_definitions(-DUSE_CRYPTO_MBEDTLS=1)
        set (MPROXY_LIBS ${MPROXY_LIBS} ${MBEDCRYPTO_LIBRARY})
    else()
        message(WARNING "Unable to find mbed TLS ")
        set(USE_CRYPTO_MBEDTLS OFF)
    endif()
endif()

if (ENABLE_SS)
    if (USE_CRYPTO_OPENSSL OR USE_CRYPTO_MBEDTLS)
        add_definitions(-DENABLE_SS=1)
        set(MPROXY_SOURCES ${MPROXY_SOURCES} ss_connector.c encrypt.c )
    else()
        message(WARNING "OpenSSL or mbed TLS is required to enable shadowsocks protocol ") 
        set(ENABLE_SS OFF)
    endif()
else()
	set(USE_CRYPTO_OPENSSL OFF)
    set(USE_CRYPTO_MBEDTLS OFF)
endif()

add_executable(mproxy ${MPROXY_SOURCES} )
target_link_libraries(mproxy evhtp ${MPROXY_LIBS})

message("")
message(STATUS "${Blue}CMAKE_BUILD_TYPE${ColourReset}           : " ${BoldRed}${CMAKE_BUILD_TYPE}${ColourReset})
message(STATUS "${Blue}ENABLE_HTTPS_PROXY${ColourReset}         : " ${BoldRed}${ENABLE_HTTPS_PROXY}${ColourReset})
message(STATUS "${Blue}ENABLE_SS${ColourReset}                  : " ${BoldRed}${ENABLE_SS}${ColourReset})
message(STATUS "${Blue}USE_CRYPTO_OPENSSL${ColourReset}         : " ${BoldRed}${USE_CRYPTO_OPENSSL}${ColourReset})
message(STATUS "${Blue}USE_CRYPTO_MBEDTLS${ColourReset}         : " ${BoldRed}${USE_CRYPTO_MBEDTLS}${ColourReset})

# vim:set sts=4 sw=4 tw=0 et:

__enable_cxx17()
set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)

set(cpr_VERSION_MAJOR 1)
set(cpr_VERSION_MINOR 9)
set(cpr_VERSION_PATCH 0)
set(CPR_VERSION "1.9.0")

math(EXPR cpr_VERSION_NUM "${cpr_VERSION_MAJOR} * 0x10000 + ${cpr_VERSION_MINOR} * 0x100 + ${cpr_VERSION_PATCH}" OUTPUT_FORMAT HEXADECIMAL)
configure_file("${CMAKE_CURRENT_SOURCE_DIR}/cmake/cprver.h.in" "${CMAKE_CURRENT_BINARY_DIR}/cpr/cprver.h")

macro(cpr_option OPTION_NAME OPTION_TEXT OPTION_DEFAULT)
    option(${OPTION_NAME} ${OPTION_TEXT} ${OPTION_DEFAULT})
    if(DEFINED ENV{${OPTION_NAME}})
        # Allow overriding the option through an environment variable
        set(${OPTION_NAME} $ENV{${OPTION_NAME}})
    endif()
    if(${OPTION_NAME})
        add_definitions(-D${OPTION_NAME})
    endif()
    message(STATUS "  ${OPTION_NAME}: ${${OPTION_NAME}}")
endmacro()

option(BUILD_SHARED_LIBS "Build libraries as shared libraries" ON)
cpr_option(CPR_ENABLE_SSL "Enables or disables the SSL backend. Required to perform HTTPS requests." ON)

include(GNUInstallDirs)
include(FetchContent)
include(cmake/code_coverage.cmake)
include(cmake/sanitizer.cmake)
include(cmake/clear_variable.cmake)

# So CMake can find FindMbedTLS.cmake
set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake;${CMAKE_MODULE_PATH}")

# Cppcheck
if(CPR_ENABLE_CPPCHECK)
    if(CPR_BUILD_TESTS OR CPR_BUILD_TESTS_SSL)
        message(FATAL_ERROR "Cppcheck is incompatible with building tests. Make sure to disable CPR_ENABLE_CPPCHECK or disable tests by setting CPR_BUILD_TESTS and CPR_BUILD_TESTS_SSL to OFF. This is because Cppcheck would try to check the google tests source code and then fail. ")
    endif()
    include(cmake/cppcheck.cmake)
endif()

if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
else()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -Wpedantic -Werror")
endif()

# SSL
if(CPR_ENABLE_SSL)
    if(CPR_FORCE_OPENSSL_BACKEND OR CPR_FORCE_WINSSL_BACKEND OR CPR_FORCE_DARWINSSL_BACKEND OR CPR_FORCE_MBEDTLS_BACKEND)
        message(STATUS "Disabled SSL backend auto detect since either CPR_FORCE_OPENSSL_BACKEND, CPR_FORCE_DARWINSSL_BACKEND, CPR_FORCE_MBEDTLS_BACKEND, or CPR_FORCE_WINSSL_BACKEND is enabled.")
        set(DETECT_SSL_BACKEND OFF CACHE INTERNAL "" FORCE)
    else()
        message(STATUS "Automatically detecting SSL backend.")
        set(DETECT_SSL_BACKEND ON CACHE INTERNAL "" FORCE)
    endif()

    if(CPR_FORCE_WINSSL_BACKEND AND (NOT WIN32))
        message(FATAL_ERROR "WinSSL is only available on Windows! Use either OpenSSL (CPR_FORCE_OPENSSL_BACKEND) or DarwinSSL (CPR_FORCE_DARWINSSL_BACKEND) instead.")
    endif()

    if(DETECT_SSL_BACKEND)
        message(STATUS "Detecting SSL backend...")
        if(WIN32)
            message(STATUS "SSL auto detect: Using WinSSL.")
            set(SSL_BACKEND_USED "WinSSL")
        elseif(APPLE)
            message(STATUS "SSL auto detect: Using DarwinSSL.")
            set(CPR_BUILD_TESTS_SSL OFF)
            set(SSL_BACKEND_USED "DarwinSSL")
        else()
            find_package(OpenSSL)
            if(OPENSSL_FOUND)
                message(STATUS "SSL auto detect: Using OpenSSL.")
                set(SSL_BACKEND_USED "OpenSSL")
            else()
                find_package(MbedTLS)
                if(MBEDTLS_FOUND)
                    set(SSL_BACKEND_USED "MbedTLS")
                else()
                    message(FATAL_ERROR "No valid SSL backend found! Please install OpenSSL, Mbed TLS or disable SSL by setting CPR_ENABLE_SSL to OFF.")
                endif()
            endif()
        endif()
    else()
        if(CPR_FORCE_OPENSSL_BACKEND)
            find_package(OpenSSL)
            if(OPENSSL_FOUND)
                message(STATUS "Using OpenSSL.")
                set(SSL_BACKEND_USED "OpenSSL")
            else()
                message(FATAL_ERROR "CPR_FORCE_OPENSSL_BACKEND enabled but we were not able to find OpenSSL!")
            endif()
        elseif(CPR_FORCE_WINSSL_BACKEND)
            message(STATUS "Using WinSSL.")
            set(SSL_BACKEND_USED "WinSSL")
        elseif(CPR_FORCE_DARWINSSL_BACKEND)
            message(STATUS "Using DarwinSSL.")
            set(CPR_BUILD_TESTS_SSL OFF)
            set(SSL_BACKEND_USED "DarwinSSL")
        elseif(CPR_FORCE_MBEDTLS_BACKEND)
            message(STATUS "Using Mbed TLS.")
            set(CPR_BUILD_TESTS_SSL OFF)
            set(SSL_BACKEND_USED "MbedTLS")
        endif()
    endif()
endif()

if(SSL_BACKEND_USED STREQUAL "OpenSSL")
# Fix missing OpenSSL includes for Windows since in 'ssl_ctx.cpp' we include OpenSSL directly
find_package(OpenSSL REQUIRED)
    add_compile_definitions(OPENSSL_BACKEND_USED)
endif()

__assert_target(curl)
__assert_target(zlib)

set(SRCS 
        cpr/accept_encoding.cpp
        cpr/async.cpp
        cpr/auth.cpp
        cpr/bearer.cpp
        cpr/cert_info.cpp
        cpr/cookies.cpp
        cpr/cprtypes.cpp
        cpr/curl_container.cpp
        cpr/curlholder.cpp
        cpr/error.cpp
        cpr/multipart.cpp
        cpr/parameters.cpp
        cpr/payload.cpp
        cpr/proxies.cpp
        cpr/proxyauth.cpp
        cpr/session.cpp
        cpr/threadpool.cpp
        cpr/timeout.cpp
        cpr/unix_socket.cpp
        cpr/util.cpp
        cpr/response.cpp
        cpr/redirect.cpp
        cpr/interceptor.cpp
        cpr/ssl_ctx.cpp
		
		include/cpr/accept_encoding.h
		include/cpr/api.h
		include/cpr/async.h
		include/cpr/auth.h
		include/cpr/bearer.h
		include/cpr/body.h
		include/cpr/buffer.h
		include/cpr/cert_info.h
		include/cpr/cookies.h
		include/cpr/cpr.h
		include/cpr/cprtypes.h
		include/cpr/curlholder.h
		include/cpr/curlholder.h
		include/cpr/error.h
		include/cpr/file.h
		include/cpr/limit_rate.h
		include/cpr/local_port.h
		include/cpr/local_port_range.h
		include/cpr/multipart.h
		include/cpr/parameters.h
		include/cpr/payload.h
		include/cpr/proxies.h
		include/cpr/proxyauth.h
		include/cpr/response.h
		include/cpr/session.h
		include/cpr/singleton.h
		include/cpr/ssl_ctx.h
		include/cpr/ssl_options.h
		include/cpr/threadpool.h
		include/cpr/timeout.h
		include/cpr/unix_socket.h
		include/cpr/util.h
		include/cpr/verbose.h
		include/cpr/interface.h
		include/cpr/redirect.h
		include/cpr/http_version.h
		include/cpr/interceptor.h
		include/cpr/filesystem.h
		${CMAKE_CURRENT_BINARY_DIR}/cpr/cprver.h
		)
		
set(INCS ${CMAKE_CURRENT_SOURCE_DIR}/include ${CMAKE_CURRENT_BINARY_DIR})
set(LIBS curl zlib)
__add_real_target(cpr dll SOURCE ${SRCS}
						  INC ${INCS}
						  LIB ${LIBS}
						  INTERFACE ${INCS}
						  )

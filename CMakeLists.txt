# CMakeLists.txt
#
# Copyright (C) 2006-2024 wolfSSL Inc.
#
# This file is part of wolfPKCS11.
#
# Usage:
# $ mkdir build
# $ cd build
# $ cmake ..
# $ cmake --build .
#
# To build with debugging use:
# $ cmake .. -DCMAKE_BUILD_TYPE=Debug

####################################################
# Project
####################################################

cmake_minimum_required(VERSION 3.16)

if(${CMAKE_VERSION} VERSION_LESS "3.22")
    message(STATUS "This project recommends using CMake version 3.22 or higher. You are using ${CMAKE_VERSION}.")
else()
    cmake_policy(SET CMP0128 NEW)
endif()

if("${CMAKE_SOURCE_DIR}" STREQUAL "${CMAKE_BINARY_DIR}")
    message(FATAL_ERROR "In-source builds are not allowed.\
     Run cmake from a separate directory from where CMakeLists.txt lives.\
     NOTE: cmake will now create CMakeCache.txt and CMakeFiles/*.\
     You must delete them, or cmake will refuse to work.")
endif()

project(wolfpkcs11 VERSION 1.3.0 LANGUAGES C)

# shared library versioning
# increment if interfaces have been removed or changed
set(WOLFPKCS11_LIBRARY_VERSION_FIRST 3)

# increment if interfaces have been added
# set to zero if WOLFPKCS11_LIBRARY_VERSION_FIRST is incremented
set(WOLFPKCS11_LIBRARY_VERSION_SECOND 1)

# increment if source code has changed
# set to zero if WOLFPKCS11_LIBRARY_VERSION_FIRST is incremented or
# WOLFPKCS11_LIBRARY_VERSION_SECOND is incremented
set(WOLFPKCS11_LIBRARY_VERSION_THIRD 0)

set(LIBTOOL_FULL_VERSION ${WOLFPKCS11_LIBRARY_VERSION_FIRST}.${WOLFPKCS11_LIBRARY_VERSION_SECOND}.${WOLFPKCS11_LIBRARY_VERSION_THIRD})

set(WOLFPKCS11_DEFINITIONS)
set(WOLFPKCS11_LINK_LIBS)
set(WOLFPKCS11_INCLUDE_DIRS)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules/")
include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/functions.cmake)

####################################################
# Compiler
####################################################
# Let CMake choose default compiler

if(CMAKE_C_COMPILER_ID STREQUAL "AppleClang")
    # Silence ranlib warning "has no symbols"
    set(CMAKE_C_ARCHIVE_CREATE   "<CMAKE_AR> Scr <TARGET> <LINK_FLAGS> <OBJECTS>")
    set(CMAKE_CXX_ARCHIVE_CREATE "<CMAKE_AR> Scr <TARGET> <LINK_FLAGS> <OBJECTS>")
    set(CMAKE_C_ARCHIVE_FINISH   "<CMAKE_RANLIB> -no_warning_for_no_symbols -c <TARGET>")
    set(CMAKE_CXX_ARCHIVE_FINISH "<CMAKE_RANLIB> -no_warning_for_no_symbols -c <TARGET>")
endif()

include(CheckFunctionExists)

# TODO: Also check if these functions are declared by the
#       expected headers. See comments around
#       AC_CHECK_FUNCS/AC_CHECK_DECLS in configure.ac.
check_function_exists("gethostbyname" HAVE_GETHOSTBYNAME)
check_function_exists("getaddrinfo" HAVE_GETADDRINFO)
check_function_exists("gettimeofday" HAVE_GETTIMEOFDAY)
check_function_exists("inet_ntoa" HAVE_INET_NTOA)
check_function_exists("memset" HAVE_MEMSET)
check_function_exists("socket" HAVE_SOCKET)

include(CheckTypeSize)

check_type_size("long long" SIZEOF_LONG_LONG)
check_type_size("long" SIZEOF_LONG)


if(CMAKE_VERSION VERSION_LESS "3.20")
    # TestBigEndian was deprecated in 3.20
    include(TestBigEndian)
    test_big_endian(IS_BIG_ENDIAN)
    set(CMAKE_C_BYTE_ORDER "LITTLE_ENDIAN")
    if(IS_BIG_ENDIAN)
        set(CMAKE_C_BYTE_ORDER "BIG_ENDIAN")
    endif()
endif()

if(DEFINED WARNING_C_FLAGS)
    set(CMAKE_C_FLAGS "${WARNING_C_FLAGS} ${CMAKE_C_FLAGS}")
elseif(WIN32)
    # Windows cl.exe does not support the -Wextra, -Wno-unused and -Werror flags.
    set(CMAKE_C_FLAGS "-Wall ${CMAKE_C_FLAGS}")
else()
    set(CMAKE_C_FLAGS "-Wall -Wextra -Wno-unused -Werror ${CMAKE_C_FLAGS}")
endif()


####################################################
# Build Options
####################################################

add_option("WOLFPKCS11_INSTALL"
    "Create install target for WolfPKCS11 project"
    "yes" "yes;no")


# Enable Debugging
add_option("WOLFPKCS11_DEBUG"
    "Enables option for debug (default: disabled)"
    "no" "yes;no")

if(WOLFPKCS11_DEBUG)
    set(CMAKE_C_FLAGS "-g ${CMAKE_C_FLAGS}")
    list(APPEND WOLFPKCS11_DEFINITIONS
        "-DDEBUG"
        "-DDEBUG_WOLFPKCS11")
# else()
    # set(CMAKE_C_FLAGS "-O2 ${CMAKE_C_FLAGS}")
    # list(APPEND WOLFPKCS11_DEFINITIONS
    #     "-DNDEBUG")
endif()


# Coverage
add_option("WOLFPKCS11_COVERAGE"
    "Enables option for coverage (default: disabled)"
    "no" "yes;no")

if(WOLFPKCS11_COVERAGE)
    set(CMAKE_C_FLAGS "--coverage ${CMAKE_C_FLAGS}")
endif()


# Single threaded
add_option("WOLFPKCS11_SINGLE_THREADED"
    "Enable wolfPKCS11 single threaded (default: disabled)"
    "no" "yes;no")

# TODO: Logic here isn't complete, yet (see AX_PTHREAD)
if(NOT WOLFPKCS11_SINGLE_THREADED)
    if(CMAKE_USE_PTHREADS_INIT)
        list(APPEND WOLFPKCS11_LINK_LIBS Threads::Threads)
        set(HAVE_PTHREAD 1)
        list(APPEND WOLFPKCS11_DEFINITIONS
            "-DHAVE_PTHREAD"
            "-D_POSIX_THREADS")
    endif()
else()
    list(APPEND WOLFPKCS11_DEFINITIONS "-DSINGLE_THREADED")
endif()


# RSA
add_option("WOLFPKCS11_RSA"
    "Enable RSA (default: enabled)"
    "yes" "yes;no")

if(NOT WOLFPKCS11_RSA)
    list(APPEND WOLFPKCS11_DEFINITIONS "-DNO_RSA")
endif()


# RSA-OAEP
add_option("WOLFPKCS11_OAEP"
    "Enable RSA OAEP (default: enabled)"
    "yes" "yes;no")

if(NOT WOLFPKCS11_RSA)
    override_cache(WOLFPKCS11_OAEP "no")
endif()
if(NOT WOLFPKCS11_OAEP)
    list(APPEND WOLFPKCS11_DEFINITIONS "-DWC_NO_RSA_OAEP")
endif()


# RSA-PSS
add_option("WOLFPKCS11_RSA_PSS"
    "Enable RSA-PSS (default: enabled)"
    "yes" "yes;no")

if(NOT WOLFPKCS11_RSA)
    override_cache(WOLFPKCS11_RSA_PSS "no")
endif()
if(WOLFPKCS11_RSA_PSS)
    list(APPEND WOLFPKCS11_DEFINITIONS "-DWC_RSA_PSS")
endif()


# Key Gen
add_option("WOLFPKCS11_KEYGEN"
    "Enable key generation (default: enabled)])"
    "yes" "yes;no")

if(WOLFPKCS11_KEYGEN)
    list(APPEND WOLFPKCS11_DEFINITIONS "-DWOLFSSL_KEY_GEN")
endif()


# ECC
add_option("WOLFPKCS11_ECC"
    "Enable ECC (default: enabled)"
    "yes" "yes;no")

if(WOLFPKCS11_ECC)
    list(APPEND WOLFPKCS11_DEFINITIONS "-DHAVE_ECC")
endif()


# ML-DSA (Dilithium)
add_option("WOLFPKCS11_DILITHIUM"
    "Enable Dilithium (default: enabled)"
    "yes" "yes;no")

if(WOLFPKCS11_DILITHIUM)
    list(APPEND WOLFPKCS11_DEFINITIONS "-DHAVE_DILITHIUM")
endif()


# DH
add_option("WOLFPKCS11_DH"
    "Enable DH (default: enabled)"
    "yes" "yes;no;const")

if(NOT WOLFPKCS11_DH)
    list(APPEND WOLFPKCS11_DEFINITIONS "-DNO_DH")
endif()


# AES
add_option("WOLFPKCS11_AES"
    "Enable AES (default: enabled)"
    "yes" "yes;no")

if(NOT WOLFPKCS11_AES)
    list(APPEND WOLFPKCS11_DEFINITIONS "-DNO_AES")
endif()


# AES-CBC
add_option("WOLFPKCS11_AESCBC"
    "Enable wolfPKCS11 AES-CBC support (default: enabled)"
    "yes" "yes;no")

if(NOT WOLFPKCS11_AESCBC)
    list(APPEND WOLFPKCS11_DEFINITIONS "-DNO_AES_CBC")
endif()


# AES-GCM
add_option("WOLFPKCS11_AESGCM"
    "Enable wolfPKCS11 AES-GCM support (default: enabled)"
    "yes" "yes;no")

if(WOLFPKCS11_AESGCM)
    list(APPEND WOLFPKCS11_DEFINITIONS "-DHAVE_AESGCM")
endif()


# HMAC
add_option("WOLFPKCS11_HMAC"
    "Enable wolfPKCS11 HMAC support (default: enabled)"
    "yes" "yes;no")

if(NOT WOLFPKCS11_HMAC)
    list(APPEND WOLFPKCS11_DEFINITIONS "-DNO_HMAC")
endif()


# MD5
add_option("WOLFPKCS11_MD5"
    "Enable MD5 (default: enabled)"
    "yes" "yes;no")

if(NOT WOLFPKCS11_MD5)
    list(APPEND WOLFPKCS11_DEFINITIONS "-DNO_MD5")
endif()


# SHA
add_option("WOLFPKCS11_SHA"
    "Enable SHA (default: enabled)"
    "yes" "yes;no")

if(NOT WOLFPKCS11_SHA)
    list(APPEND WOLFPKCS11_DEFINITIONS "-DNO_SHA")
endif()


# SHA224
add_option("WOLFPKCS11_SHA224"
    "Enable wolfPKCS11 SHA-224 support (default: enabled)"
    "yes" "yes;no")

if(WOLFPKCS11_SHA224)
    list(APPEND WOLFPKCS11_DEFINITIONS "-DWOLFSSL_SHA224")
endif()


# SHA256
add_option("WOLFPKCS11_SHA256"
    "Enable wolfPKCS11 SHA-256 support (default: enabled)"
    "yes" "yes;no")

if(NOT WOLFPKCS11_SHA256)
    list(APPEND WOLFPKCS11_DEFINITIONS "-DNO_SHA256")
endif()


# SHA384
add_option("WOLFPKCS11_SHA384"
    "Enable wolfPKCS11 SHA-384 support (default: enabled)"
    "yes" "yes;no")

if(WOLFPKCS11_SHA384)
    list(APPEND WOLFPKCS11_DEFINITIONS "-DWOLFSSL_SHA384")
endif()


# SHA512
add_option("WOLFPKCS11_SHA512"
    "Enable wolfPKCS11 SHA-512 support (default: enabled)"
    "yes" "yes;no")

if(WOLFPKCS11_SHA512)
    list(APPEND WOLFPKCS11_DEFINITIONS "-DWOLFSSL_SHA512")
endif()


# WolfTPM
add_option("WOLFPKCS11_TPM"
    "Enable wolfTPM keystore support (default: disabled)"
    "no" "yes;no")

if(WOLFPKCS11_TPM)
    list(APPEND WOLFPKCS11_LINK_LIBS "-lwolftpm")
    list(APPEND WOLFPKCS11_DEFINITIONS "-DWOLFPKCS11_TPM")
endif()

# If wolfpkcs11/options.h exists, delete it to avoid
# a mixup with build/wolfpkcs11/options.h.
if (EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/wolfpkcs11/options.h")
    file(REMOVE "${CMAKE_CURRENT_SOURCE_DIR}/wolfpkcs11/options.h")
endif()

# Generate user options header
message(STATUS "Generating user options header...")
if (${CMAKE_DISABLE_SOURCE_CHANGES})
    set(WOLFPKCS11_BUILD_OUT_OF_TREE_DEFAULT "${CMAKE_DISABLE_SOURCE_CHANGES}")
else()
    set(WOLFPKCS11_BUILD_OUT_OF_TREE_DEFAULT "yes")
endif()
add_option("WOLFPKCS11_BUILD_OUT_OF_TREE"
    "Don't generate files in the source tree (default: ${WOLFPKCS11_BUILD_OUT_OF_TREE_DEFAULT})"
    "${WOLFPKCS11_BUILD_OUT_OF_TREE_DEFAULT}" "yes;no")
if (${WOLFPKCS11_BUILD_OUT_OF_TREE})
   set(WOLFPKCS11_OUTPUT_BASE ${CMAKE_CURRENT_BINARY_DIR})
else()
   set(WOLFPKCS11_OUTPUT_BASE ${CMAKE_CURRENT_SOURCE_DIR})
endif()
set(OPTION_FILE "${WOLFPKCS11_OUTPUT_BASE}/wolfpkcs11/options.h")

file(REMOVE ${OPTION_FILE})


####################################################
# Library Target
####################################################

# TODO: - Build shared/static libs based on enables. Check CMake
#         global flag BUILD_SHARED_LIBS.
option(BUILD_SHARED_LIBS
    "Build shared libraries (.dll/.so) instead of static ones (.lib/.a)"
    ON)

set(LIB_SOURCES
    src/internal.c
    src/wolfpkcs11.c
    src/slot.c
    src/crypto.c
)
if(BUILD_SHARED_LIBS)
    message(STATUS "BUILD_SHARED_LIBS enabled: ${LIB_SOURCES}")
    add_library(wolfpkcs11 SHARED ${LIB_SOURCES})
else()
    message(STATUS "Static Libs: ${LIB_SOURCES}")
    add_library(wolfpkcs11 STATIC ${LIB_SOURCES})
    list(APPEND WOLFPKCS11_DEFINITIONS "-DHAVE_PKCS11_STATIC")
endif()

add_library(wolfpkcs11::wolfpkcs11 ALIAS wolfpkcs11)

foreach(DEF IN LISTS WOLFPKCS11_DEFINITIONS)
    string(REGEX MATCH "^(-D)?([^=]+)(=(.*))?$" DEF_MATCH ${DEF})
    if (NOT "${CMAKE_MATCH_4}" STREQUAL "")
        set(${CMAKE_MATCH_2} ${CMAKE_MATCH_4})
        # message("set(${CMAKE_MATCH_2} ${CMAKE_MATCH_4})")
    else()
        set(${CMAKE_MATCH_2} 1)
        # message("set(${CMAKE_MATCH_2} 1)")
    endif()
endforeach()

# If new build options are added please update the cmake/options.h.in
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/cmake/options.h.in ${OPTION_FILE})

set_target_properties(wolfpkcs11
    PROPERTIES
        SOVERSION ${WOLFPKCS11_LIBRARY_VERSION_FIRST}
        VERSION ${LIBTOOL_FULL_VERSION}
)

target_compile_definitions(wolfpkcs11 PRIVATE "BUILDING_WOLFPKCS11")
target_compile_definitions(wolfpkcs11 PUBLIC ${WOLFPKCS11_DEFINITIONS})


####################################################
# Include Directories
####################################################

target_include_directories(wolfpkcs11
    PUBLIC
        $<INSTALL_INTERFACE:include>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
)


####################################################
# Link Libraries
####################################################

list(APPEND WOLFPKCS11_LINK_LIBS "wolfssl")
target_link_libraries(wolfpkcs11 PUBLIC ${WOLFPKCS11_LINK_LIBS})

if(WIN32)
    # For Windows link ws2_32
    target_link_libraries(wolfpkcs11 PUBLIC
        $<$<PLATFORM_ID:Windows>:ws2_32 crypt32>)
else()
    target_link_libraries(wolfpkcs11
        PUBLIC m)
endif()


####################################################
# Tests
####################################################

enable_testing()
# ToDo: add tests

####################################################
# Installation
####################################################

include(GNUInstallDirs)

set(HEADER_EXCLUDE
    "internal.h"
    "options.h"
    "store.h"
    "version.h"
    "visibility.h"
  )

list(JOIN HEADER_EXCLUDE "|" EXCLUDED_HEADERS_REGEX)

string(PREPEND EXCLUDED_HEADERS_REGEX "(")
string(APPEND  EXCLUDED_HEADERS_REGEX ")")

if(WOLFPKCS11_INSTALL)
    # Install the library
    install(TARGETS wolfpkcs11
            EXPORT wolfpkcs11-targets
            LIBRARY DESTINATION lib
            ARCHIVE DESTINATION lib
            RUNTIME DESTINATION bin
            )
    # Install the headers
    install(DIRECTORY ${WOLFPKCS11_OUTPUT_BASE}/wolfpkcs11/
            DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/wolfpkcs11
            FILES_MATCHING PATTERN "*.h"
            REGEX ${EXCLUDED_HEADERS_REGEX} EXCLUDE)
    install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/wolfpkcs11/
            DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/wolfpkcs11
            FILES_MATCHING PATTERN "*.h"
            REGEX ${EXCLUDED_HEADERS_REGEX} EXCLUDE)
endif()

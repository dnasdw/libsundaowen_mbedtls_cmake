cmake_minimum_required(VERSION 2.6)
project("mbed TLS" C)

option(USE_PKCS11_HELPER_LIBRARY "Build mbed TLS with the pkcs11-helper library." OFF)
option(ENABLE_ZLIB_SUPPORT "Build mbed TLS with zlib library." OFF)

option(UNSAFE_BUILD "Allow unsafe builds. These builds ARE NOT SECURE." OFF)

string(REGEX MATCH "Clang" CMAKE_COMPILER_IS_CLANG "${CMAKE_C_COMPILER_ID}")
string(REGEX MATCH "GNU" CMAKE_COMPILER_IS_GNU "${CMAKE_C_COMPILER_ID}")
string(REGEX MATCH "IAR" CMAKE_COMPILER_IS_IAR "${CMAKE_C_COMPILER_ID}")
string(REGEX MATCH "MSVC" CMAKE_COMPILER_IS_MSVC "${CMAKE_C_COMPILER_ID}")

# Warning string - created as a list for compatibility with CMake 2.8
set(WARNING_BORDER "*******************************************************\n")
set(NULL_ENTROPY_WARN_L1 "****  WARNING!  MBEDTLS_TEST_NULL_ENTROPY defined!\n")
set(NULL_ENTROPY_WARN_L2 "****  THIS BUILD HAS NO DEFINED ENTROPY SOURCES\n")
set(NULL_ENTROPY_WARN_L3 "****  AND IS *NOT* SUITABLE FOR PRODUCTION USE\n")

set(NULL_ENTROPY_WARNING "${WARNING_BORDER}"
                         "${NULL_ENTROPY_WARN_L1}"
                         "${NULL_ENTROPY_WARN_L2}"
                         "${NULL_ENTROPY_WARN_L3}"
                         "${WARNING_BORDER}")

set(CTR_DRBG_128_BIT_KEY_WARN_L1 "****  WARNING!  MBEDTLS_CTR_DRBG_USE_128_BIT_KEY defined!\n")
set(CTR_DRBG_128_BIT_KEY_WARN_L2 "****  Using 128-bit keys for CTR_DRBG limits the security of generated\n")
set(CTR_DRBG_128_BIT_KEY_WARN_L3 "****  keys and operations that use random values generated to 128-bit security\n")

set(CTR_DRBG_128_BIT_KEY_WARNING "${WARNING_BORDER}"
                         "${CTR_DRBG_128_BIT_KEY_WARN_L1}"
                         "${CTR_DRBG_128_BIT_KEY_WARN_L2}"
                         "${CTR_DRBG_128_BIT_KEY_WARN_L3}"
                         "${WARNING_BORDER}")

string(REGEX MATCH "Clang" CMAKE_COMPILER_IS_CLANG "${CMAKE_C_COMPILER_ID}")

if(CMAKE_COMPILER_IS_GNU)
    # some warnings we want are not available with old GCC versions
    # note: starting with CMake 2.8 we could use CMAKE_C_COMPILER_VERSION
    execute_process(COMMAND ${CMAKE_C_COMPILER} -dumpversion
                    OUTPUT_VARIABLE GCC_VERSION)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -W -Wdeclaration-after-statement -Wwrite-strings")
    if (GCC_VERSION VERSION_GREATER 4.5 OR GCC_VERSION VERSION_EQUAL 4.5)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wlogical-op")
    endif()
    if (GCC_VERSION VERSION_GREATER 4.8 OR GCC_VERSION VERSION_EQUAL 4.8)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wshadow")
    endif()
    set(CMAKE_C_FLAGS_RELEASE     "-O2")
    set(CMAKE_C_FLAGS_DEBUG       "-O0 -g3")
    set(CMAKE_C_FLAGS_COVERAGE    "-O0 -g3 --coverage")
    set(CMAKE_C_FLAGS_ASAN        "-Werror -fsanitize=address -fno-common -O3")
    set(CMAKE_C_FLAGS_ASANDBG     "-Werror -fsanitize=address -fno-common -O1 -g3 -fno-omit-frame-pointer -fno-optimize-sibling-calls ")
    set(CMAKE_C_FLAGS_CHECK       "-Werror -Os")
    set(CMAKE_C_FLAGS_CHECKFULL   "${CMAKE_C_FLAGS_CHECK} -Wcast-qual")
endif(CMAKE_COMPILER_IS_GNU)

if(CMAKE_COMPILER_IS_CLANG)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -W -Wdeclaration-after-statement -Wwrite-strings -Wpointer-arith -Wimplicit-fallthrough -Wshadow")
    set(CMAKE_C_FLAGS_RELEASE     "-O2")
    set(CMAKE_C_FLAGS_DEBUG       "-O0 -g3")
    set(CMAKE_C_FLAGS_COVERAGE    "-O0 -g3 --coverage")
    set(CMAKE_C_FLAGS_ASAN        "-Werror -fsanitize=address -fno-common -fsanitize=undefined -fno-sanitize-recover=all -O3")
    set(CMAKE_C_FLAGS_ASANDBG     "-Werror -fsanitize=address -fno-common -fsanitize=undefined -fno-sanitize-recover=all -O1 -g3 -fno-omit-frame-pointer -fno-optimize-sibling-calls ")
    set(CMAKE_C_FLAGS_MEMSAN      "-Werror -fsanitize=memory -O3")
    set(CMAKE_C_FLAGS_MEMSANDBG   "-Werror -fsanitize=memory -O1 -g3 -fno-omit-frame-pointer -fno-optimize-sibling-calls -fsanitize-memory-track-origins=2")
    set(CMAKE_C_FLAGS_CHECK       "-Werror -Os")
endif(CMAKE_COMPILER_IS_CLANG)

if(CMAKE_COMPILER_IS_IAR)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} --warn_about_c_style_casts --warnings_are_errors -Ohz")
endif(CMAKE_COMPILER_IS_IAR)

if(CMAKE_COMPILER_IS_MSVC)
    # Strictest warnings, and treat as errors
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /W3")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /WX")
endif(CMAKE_COMPILER_IS_MSVC)

if(CMAKE_BUILD_TYPE STREQUAL "Coverage")
    if(CMAKE_COMPILER_IS_GNU OR CMAKE_COMPILER_IS_CLANG)
        set(CMAKE_SHARED_LINKER_FLAGS "--coverage")
    endif(CMAKE_COMPILER_IS_GNU OR CMAKE_COMPILER_IS_CLANG)
endif(CMAKE_BUILD_TYPE STREQUAL "Coverage")

include_directories("${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/include/")

if(ENABLE_ZLIB_SUPPORT)
    find_package(ZLIB)

    if(ZLIB_FOUND)
        include_directories(${ZLIB_INCLUDE_DIR})
    endif(ZLIB_FOUND)
endif(ENABLE_ZLIB_SUPPORT)

if(MSVC)
    option(USE_MSVC_MT_FOR_STATIC_MBEDTLS_LIBRARY "" OFF)
endif(MSVC)

option(USE_STATIC_MBEDTLS_LIBRARY "Build mbed TLS static library." ON)
option(USE_SHARED_MBEDTLS_LIBRARY "Build mbed TLS shared library." OFF)
option(LINK_WITH_PTHREAD "Explicitly link mbed TLS library to pthread." OFF)

set(src_crypto
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/aes.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/aesni.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/arc4.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/aria.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/asn1parse.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/asn1write.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/base64.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/bignum.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/blowfish.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/camellia.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/ccm.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/chacha20.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/chachapoly.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/cipher.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/cipher_wrap.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/cmac.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/ctr_drbg.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/des.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/dhm.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/ecdh.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/ecdsa.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/ecjpake.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/ecp.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/ecp_curves.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/entropy.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/entropy_poll.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/error.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/gcm.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/havege.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/hkdf.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/hmac_drbg.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/md.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/md2.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/md4.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/md5.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/md_wrap.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/memory_buffer_alloc.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/nist_kw.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/oid.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/padlock.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/pem.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/pk.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/pk_wrap.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/pkcs12.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/pkcs5.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/pkparse.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/pkwrite.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/platform.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/platform_util.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/poly1305.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/ripemd160.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/rsa.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/rsa_internal.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/sha1.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/sha256.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/sha512.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/threading.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/timing.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/version.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/version_features.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/xtea.c"
)

set(src_x509
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/certs.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/pkcs11.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/x509.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/x509_create.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/x509_crl.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/x509_crt.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/x509_csr.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/x509write_crt.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/x509write_csr.c"
)

set(src_tls
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/debug.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/net_sockets.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/ssl_cache.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/ssl_ciphersuites.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/ssl_cli.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/ssl_cookie.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/ssl_srv.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/ssl_ticket.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/mbedtls/library/ssl_tls.c"
)

if(CMAKE_COMPILER_IS_GNUCC)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wmissing-declarations -Wmissing-prototypes")
endif(CMAKE_COMPILER_IS_GNUCC)

if(CMAKE_COMPILER_IS_CLANG)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wmissing-declarations -Wmissing-prototypes -Wdocumentation -Wno-documentation-deprecated-sync -Wunreachable-code")
endif(CMAKE_COMPILER_IS_CLANG)

if(UNSAFE_BUILD)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-error")
    set(CMAKE_C_FLAGS_ASAN "${CMAKE_C_FLAGS_ASAN} -Wno-error")
    set(CMAKE_C_FLAGS_ASANDBG "${CMAKE_C_FLAGS_ASANDBG} -Wno-error")
endif(UNSAFE_BUILD)

if(WIN32)
    set(libs ${libs} ws2_32)
endif(WIN32)

if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    SET(CMAKE_C_ARCHIVE_CREATE   "<CMAKE_AR> Scr <TARGET> <LINK_FLAGS> <OBJECTS>")
    SET(CMAKE_CXX_ARCHIVE_CREATE "<CMAKE_AR> Scr <TARGET> <LINK_FLAGS> <OBJECTS>")
    SET(CMAKE_C_ARCHIVE_FINISH   "<CMAKE_RANLIB> -no_warning_for_no_symbols -c <TARGET>")
    SET(CMAKE_CXX_ARCHIVE_FINISH "<CMAKE_RANLIB> -no_warning_for_no_symbols -c <TARGET>")
endif()

if(HAIKU)
    set(libs ${libs} network)
endif(HAIKU)

if(USE_PKCS11_HELPER_LIBRARY)
    set(libs ${libs} pkcs11-helper)
endif(USE_PKCS11_HELPER_LIBRARY)

if(ENABLE_ZLIB_SUPPORT)
    set(libs ${libs} ${ZLIB_LIBRARIES})
endif(ENABLE_ZLIB_SUPPORT)

if(LINK_WITH_PTHREAD)
    set(libs ${libs} pthread)
endif()

if (NOT USE_STATIC_MBEDTLS_LIBRARY AND NOT USE_SHARED_MBEDTLS_LIBRARY)
    message(FATAL_ERROR "Need to choose static or shared mbedtls build!")
endif(NOT USE_STATIC_MBEDTLS_LIBRARY AND NOT USE_SHARED_MBEDTLS_LIBRARY)

if(USE_STATIC_MBEDTLS_LIBRARY AND USE_SHARED_MBEDTLS_LIBRARY)
    set(mbedtls_static_target "mbedtls_static")
    set(mbedx509_static_target "mbedx509_static")
    set(mbedcrypto_static_target "mbedcrypto_static")
elseif(USE_STATIC_MBEDTLS_LIBRARY)
    set(mbedtls_static_target "mbedtls")
    set(mbedx509_static_target "mbedx509")
    set(mbedcrypto_static_target "mbedcrypto")
endif()

if(USE_STATIC_MBEDTLS_LIBRARY)
    if(MSVC AND USE_MSVC_MT_FOR_STATIC_MBEDTLS_LIBRARY)
        string(REPLACE "/MDd" "" CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG}")
        set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} /MTd")
        string(REPLACE "/MD" "" CMAKE_C_FLAGS_MINSIZEREL "${CMAKE_C_FLAGS_MINSIZEREL}")
        set(CMAKE_C_FLAGS_MINSIZEREL "${CMAKE_C_FLAGS_MINSIZEREL} /MT")
        string(REPLACE "/MD" "" CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE}")
        set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /MT")
        string(REPLACE "/MD" "" CMAKE_C_FLAGS_RELWITHDEBINFO "${CMAKE_C_FLAGS_RELWITHDEBINFO}")
        set(CMAKE_C_FLAGS_RELWITHDEBINFO "${CMAKE_C_FLAGS_RELWITHDEBINFO} /MT")
    endif()

    add_library(${mbedcrypto_static_target} STATIC ${src_crypto})
    set_target_properties(${mbedcrypto_static_target} PROPERTIES OUTPUT_NAME mbedcrypto)
    target_link_libraries(${mbedcrypto_static_target} ${libs})

    add_library(${mbedx509_static_target} STATIC ${src_x509})
    set_target_properties(${mbedx509_static_target} PROPERTIES OUTPUT_NAME mbedx509)
    target_link_libraries(${mbedx509_static_target} ${libs} ${mbedcrypto_static_target})

    add_library(${mbedtls_static_target} STATIC ${src_tls})
    set_target_properties(${mbedtls_static_target} PROPERTIES OUTPUT_NAME mbedtls)
    target_link_libraries(${mbedtls_static_target} ${libs} ${mbedx509_static_target})
endif(USE_STATIC_MBEDTLS_LIBRARY)

if(USE_SHARED_MBEDTLS_LIBRARY)
    add_library(mbedcrypto SHARED ${src_crypto})
    set_target_properties(mbedcrypto PROPERTIES VERSION 2.16.3 SOVERSION 3)
    target_link_libraries(mbedcrypto ${libs})

    add_library(mbedx509 SHARED ${src_x509})
    set_target_properties(mbedx509 PROPERTIES VERSION 2.16.3 SOVERSION 0)
    target_link_libraries(mbedx509 ${libs} mbedcrypto)

    add_library(mbedtls SHARED ${src_tls})
    set_target_properties(mbedtls PROPERTIES VERSION 2.16.3 SOVERSION 12)
    target_link_libraries(mbedtls ${libs} mbedx509)
endif(USE_SHARED_MBEDTLS_LIBRARY)

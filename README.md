# libsundaowen_mbedtls_cmake

## example

### git
~~~
git submodule add -- "https://github.com/dnasdw/libsundaowen_mbedtls_cmake.git" "dep/mbedtls_cmake"
git submodule update --init --recursive "dep/mbedtls_cmake"
~~~

### CMakeLists.txt
~~~
#...
set(ROOT_SOURCE_DIR "${PROJECT_SOURCE_DIR}")
#...
include_directories("${ROOT_SOURCE_DIR}/dep/mbedtls_cmake/mbedtls/include")
#...
if(MSVC)
  #set(USE_MSVC_MT_FOR_STATIC_MBEDTLS_LIBRARY ON CACHE BOOL "" FORCE)
endif()
add_subdirectory("${ROOT_SOURCE_DIR}/dep/mbedtls_cmake")
#...
target_link_libraries(MY_APP_NAME mbedtls)
#...
~~~

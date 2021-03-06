cmake_minimum_required(VERSION 3.4)

project(tjs LANGUAGES C)

include(GNUInstallDirs)

if (NOT CMAKE_BUILD_TYPE)
    message(STATUS "No build type selected, default to Release")
    set(CMAKE_BUILD_TYPE "Release")
endif()
message(STATUS "Building in ${CMAKE_BUILD_TYPE} mode")
message(STATUS "Building with ${CMAKE_C_COMPILER_ID} ${CMAKE_C_COMPILER_VERSION} on ${CMAKE_SYSTEM}")

set(TJS__VERSION_MAJOR 19)
set(TJS__VERSION_MINOR 0)
set(TJS__VERSION_PATCH 0)
set(TJS__VERSION_SUFFIX "-alpha")
configure_file(
    "${PROJECT_SOURCE_DIR}/src/version.h.in"
    "${PROJECT_SOURCE_DIR}/src/version.h"
)


macro(cpr_option OPTION_NAME OPTION_TEXT OPTION_DEFAULT)
    option(${OPTION_NAME} ${OPTION_TEXT} ${OPTION_DEFAULT})
    if(DEFINED ENV{${OPTION_NAME}})
        # Allow setting the option through an environment variable
        set(${OPTION_NAME} $ENV{${OPTION_NAME}})
    endif()
    if(${OPTION_NAME})
        add_definitions(-D${OPTION_NAME})
    endif()
    message(STATUS "  ${OPTION_NAME}: ${${OPTION_NAME}}")
endmacro()


if(OSX)
    include(${PROJECT_SOURCE_DIR}/src/js/CMakeLists.txt)
endif()

#add_subdirectory(deps/quickjs EXCLUDE_FROM_ALL)

option(libuv_buildtests "" OFF)
add_subdirectory(deps/libuv EXCLUDE_FROM_ALL)

cpr_option(DISABLE_WASM "If ON, WASM support will be disabled" OFF)
if (NOT DISABLE_WASM)
    option(BUILD_WASI_SUPPORT "" ON)
    add_subdirectory(deps/wasm3/source EXCLUDE_FROM_ALL)
    target_compile_definitions(m3 PRIVATE
        d_m3LogOutput=0
    )
endif()

cpr_option(DISABLE_CURL "If ON, curl support will be disabled" OFF)
cpr_option(USE_SYSTEM_CURL "If ON, this project will look in the system paths for an installed curl library" ON)

if(NOT DISABLE_CURL)
    if(USE_SYSTEM_CURL)
        find_package(CURL)
    endif()
    if(NOT USE_SYSTEM_CURL OR NOT CURL_FOUND)
        message(STATUS "Not using system Curl, using built-in curl project instead.")
        option(HTTP_ONLY "disables all protocols except HTTP" ON)
        option(BUILD_TESTING "Set to ON to build cURL tests." OFF)
        option(BUILD_CURL_EXE "Set to ON to build cURL executable." OFF)
        add_subdirectory(deps/curl)
        set(CURL_FOUND TRUE)
        set(CURL_LIBRARIES libcurl)
        set(CURL_INCLUDE_DIRS
            ${CURL_SOURCE_DIR}/include
            ${CURL_BINARY_DIR}/include/curl)
    endif()
endif()

#add_executable(tjs
add_library(tjs
    STATIC
    src/bootstrap.c
    src/cli.c
    src/curl-utils.c
    src/dns.c
    src/error.c
    src/fs.c
    src/misc.c
    src/modules.c
    src/process.c
    src/std.c
    src/streams.c
    src/signals.c
    src/timers.c
    src/udp.c
    src/utils.c
    src/version.c
    src/vm.c
    src/wasm.c
    src/worker.c
    src/xhr.c
    src/prebuilt-js.c
#    ${CMAKE_CURRENT_BINARY_DIR}/js.c
#    ../deps/quickjs/src/cutils.c
)

set_target_properties(tjs PROPERTIES
    C_STANDARD 11
    C_STANDARD_REQUIRED ON
)

if (CURL_FOUND)
    if (USE_SYSTEM_CURL)
        target_compile_definitions(tjs PRIVATE
            TJS_HAVE_SYSTEM_CURL
        )
    endif()
    target_compile_definitions(tjs PRIVATE
        TJS_HAVE_CURL
    )
    target_include_directories(tjs PRIVATE ${CURL_INCLUDE_DIRS})
    target_link_libraries(tjs ${CURL_LIBRARIES})
endif()

string(TOLOWER ${CMAKE_SYSTEM_NAME} TJS_PLATFORM)
target_compile_definitions(tjs PRIVATE TJS__PLATFORM="${TJS_PLATFORM}")

if (NOT DISABLE_WASM)
    target_compile_definitions(tjs PRIVATE
        TJS_HAVE_WASM
    )
    target_link_libraries(tjs m3)
endif()

target_link_libraries(tjs uv_a)
if(NOT WIN32)
    target_link_libraries(tjs m)
endif()

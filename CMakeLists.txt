cmake_minimum_required(VERSION 3.21)

# Avoid warning about DOWNLOAD_EXTRACT_TIMESTAMP in CMake 3.24
if (CMAKE_VERSION VERSION_GREATER_EQUAL "3.24.0")
    cmake_policy(SET CMP0135 NEW)
endif()

include(FetchContent)

# Modules declaration
FetchContent_Declare(CMakeExtensions
    GIT_REPOSITORY https://github.com/BabylonJS/CMakeExtensions.git
    GIT_TAG ea28b7689530bfdc4905806f27ecf7e8ed4b5419)
FetchContent_Declare(ios-cmake
    GIT_REPOSITORY https://github.com/leetal/ios-cmake.git
    GIT_TAG 4.4.1)
FetchContent_Declare(arcana.cpp
    GIT_REPOSITORY https://github.com/microsoft/arcana.cpp.git
    GIT_TAG f2757396e80bc4169f2ddb938ce25367a98ffdd0)
FetchContent_Declare(UrlLib
    GIT_REPOSITORY https://github.com/BabylonJS/UrlLib.git
    GIT_TAG 59917f32f6ddfa26af07dd981842c51ce02dafcd)
FetchContent_Declare(asio
    GIT_REPOSITORY https://github.com/chriskohlhoff/asio.git
    GIT_TAG f693a3eb7fe72a5f19b975289afc4f437d373d9c)
FetchContent_Declare(llhttp
    URL "https://github.com/nodejs/llhttp/archive/refs/tags/release/v8.1.0.tar.gz")
FetchContent_Declare(googletest
    URL "https://github.com/google/googletest/archive/refs/tags/v1.14.0.tar.gz")

set(CONTENT_TO_FETCH CMakeExtensions googletest)

if(IOS)
    FetchContent_MakeAvailable(ios-cmake)
    set(CMAKE_TOOLCHAIN_FILE "${ios-cmake_SOURCE_DIR}/ios.toolchain.cmake" CACHE PATH "")
    set(PLATFORM "OS64COMBINED" CACHE STRING "")
    set(DEPLOYMENT_TARGET "12" CACHE STRING "")
endif()

# Project
project(JsRuntimeHost)

set_property(GLOBAL PROPERTY USE_FOLDERS ON)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Options

## Core
option(JSRUNTIMEHOST_CORE_APPRUNTIME "Include JsRuntimeHost Core AppRuntime" ON)
option(JSRUNTIMEHOST_CORE_APPRUNTIME_V8_INSPECTOR "Include the V8 inspector protocol required to debug JavaScript with a V8 debugger." ON)
option(JSRUNTIMEHOST_CORE_SCRIPTLOADER "Include JsRuntimeHost Core ScriptLoader" ON)

## Polyfills
option(JSRUNTIMEHOST_POLYFILL_CONSOLE "Include JsRuntimeHost Polyfill Console." ON)
option(JSRUNTIMEHOST_POLYFILL_SCHEDULING "Include JsRuntimeHost Polyfill Scheduling." ON)
option(JSRUNTIMEHOST_POLYFILL_XMLHTTPREQUEST "Include JsRuntimeHost Polyfill XMLHttpRequest." ON)
option(JSRUNTIMEHOST_POLYFILL_URL "Include JsRuntimeHost Polyfill URL and URLSearchParams." ON)
option(JSRUNTIMEHOST_POLYFILL_ABORT_CONTROLLER "Include JsRuntimeHost Polyfills AbortController and AbortSignal." ON)
option(JSRUNTIMEHOST_POLYFILL_WEBSOCKET "Include JsRuntimeHost Polyfill WebSocket." ON)

## Tests
option(JSRUNTIMEHOST_TESTS "Include JsRuntimeHost Tests." ${PROJECT_IS_TOP_LEVEL})

## Node-API
option(NAPI_BUILD_ABI "Build the ABI layer." ON)

# Dependencies

if(NOT TARGET arcana)
    set(CONTENT_TO_FETCH ${CONTENT_TO_FETCH} arcana.cpp)
endif()

if(NOT TARGET UrlLib AND JSRUNTIMEHOST_POLYFILL_XMLHTTPREQUEST)
    set(CONTENT_TO_FETCH ${CONTENT_TO_FETCH} UrlLib)
endif()

if(NAPI_JAVASCRIPT_ENGINE STREQUAL "V8" AND JSRUNTIMEHOST_CORE_APPRUNTIME_V8_INSPECTOR)
    if(NOT TARGET asio)
        set(CONTENT_TO_FETCH ${CONTENT_TO_FETCH} asio)
    endif()

    if(NOT TARGET llhttp_static)
        set(BUILD_SHARED_LIBS OFF CACHE INTERNAL "")
        set(BUILD_STATIC_LIBS ON CACHE INTERNAL "")
        set(CONTENT_TO_FETCH ${CONTENT_TO_FETCH} llhttp)
    endif()
endif()

# Fetching content
message(STATUS "Fetching dependencies for ${PROJECT_NAME} (${CONTENT_TO_FETCH})")
FetchContent_MakeAvailable(${CONTENT_TO_FETCH})
message(STATUS "Fetching dependencies for ${PROJECT_NAME} - done")

# Set properties after fetching content
if(TARGET arcana)
    set_property(TARGET arcana PROPERTY FOLDER Dependencies)
endif()

if(TARGET UrlLib AND JSRUNTIMEHOST_POLYFILL_XMLHTTPREQUEST)
    set_property(TARGET UrlLib PROPERTY FOLDER Dependencies)
endif()

if(NAPI_JAVASCRIPT_ENGINE STREQUAL "V8" AND JSRUNTIMEHOST_CORE_APPRUNTIME_V8_INSPECTOR)
    if(NOT TARGET asio)
        add_library(asio INTERFACE)
        target_include_directories(asio INTERFACE "${asio_SOURCE_DIR}/asio/include")
        set_property(TARGET asio PROPERTY FOLDER Dependencies)
    endif()
    if(NOT TARGET llhttp_static)
        set_property(TARGET llhttp_static PROPERTY FOLDER Dependencies)
    endif()
endif()

# Subdirectories

if(ANDROID)
    set(JSRUNTIMEHOST_PLATFORM "Android")
elseif(IOS)
    set(JSRUNTIMEHOST_PLATFORM "iOS")
elseif(APPLE)
    set(JSRUNTIMEHOST_PLATFORM "macOS")
elseif(WINDOWS_STORE)
    set(JSRUNTIMEHOST_PLATFORM "UWP")
elseif(WIN32)
    set(JSRUNTIMEHOST_PLATFORM "Win32")
elseif(UNIX)
    set(JSRUNTIMEHOST_PLATFORM "Unix")
else()
    message(FATAL_ERROR "Unrecognized platform: ${CMAKE_SYSTEM_NAME}")
endif()

add_subdirectory(Core)
add_subdirectory(Polyfills)

if(JSRUNTIMEHOST_TESTS)
    add_subdirectory(Tests)
endif()

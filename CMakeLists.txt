cmake_minimum_required(VERSION 3.22)
project(PaperDollTouch LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# Portable core tests run without an Android SDK/NDK or network access.
if(NOT ANDROID)
    enable_testing()
    add_executable(pdt_cursor_engine_tests
        src/core/CursorEngine.cpp
        tests/test_cursor_engine.cpp)
    target_include_directories(pdt_cursor_engine_tests PRIVATE src)
    if(CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU")
        target_compile_options(pdt_cursor_engine_tests PRIVATE -Wall -Wextra -Wpedantic -Werror)
    endif()
    add_test(NAME pdt_cursor_engine_tests COMMAND pdt_cursor_engine_tests)
    message(STATUS "Host mode: building portable cursor-core tests only. Use an Android NDK toolchain to build the mod.")
    return()
endif()

set(MOD_ID "PaperDollTouch" CACHE STRING "LeviLauncher mod package directory name")
set(MOD_NAME "PaperDollTouch (Experimental Cursor Core)" CACHE STRING "Mod display name")
set(MOD_AUTHOR "PaperDollTouch" CACHE STRING "Mod author")
set(MOD_VERSION "0.2.0" CACHE STRING "Mod version")
set(MOD_LIBRARY_NAME "PaperDollTouch" CACHE STRING "Native library name without lib prefix or .so suffix")
set(MOD_MINECRAFT_VERSIONS "[\"1.26.60\"]" CACHE STRING "Manifest minecraft_versions JSON array")

if(NOT CMAKE_ANDROID_ARCH_ABI STREQUAL "arm64-v8a")
    message(FATAL_ERROR "This initial build target is ARM64 only; got '${CMAKE_ANDROID_ARCH_ABI}'.")
endif()

include(FetchContent)
FetchContent_Declare(
    preloader_android
    GIT_REPOSITORY https://github.com/LiteLDev/preloader-android.git
    GIT_TAG 0.2.2
    SOURCE_SUBDIR cmake/preloader-sdk)
FetchContent_Declare(
    nlohmann_json
    GIT_REPOSITORY https://github.com/nlohmann/json.git
    GIT_TAG v3.11.3)
FetchContent_Declare(
    boost_pfr
    GIT_REPOSITORY https://github.com/boostorg/pfr.git
    GIT_TAG boost-1.88.0)
FetchContent_Declare(
    magic_enum
    GIT_REPOSITORY https://github.com/Neargye/magic_enum.git
    GIT_TAG v0.9.7)
FetchContent_Declare(
    fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG 11.2.0)
set(FMT_DOC OFF CACHE BOOL "" FORCE)
set(FMT_TEST OFF CACHE BOOL "" FORCE)
set(FMT_INSTALL OFF CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(preloader_android nlohmann_json boost_pfr magic_enum fmt)
FetchContent_GetProperties(preloader_android)
set(LEVI_PRELOADER_INCLUDE_DIR "${preloader_android_SOURCE_DIR}/include")

option(LEVI_LINK_PRELOADER "Build and link the public preloader SDK target" ON)
if(LEVI_LINK_PRELOADER AND NOT TARGET preloader)
    add_subdirectory("${preloader_android_SOURCE_DIR}" "${CMAKE_BINARY_DIR}/_preloader_android")
endif()

add_library(${MOD_LIBRARY_NAME} SHARED
    src/main.cpp
    src/core/CursorEngine.cpp
    src/mod/PaperDollTouchMod.cpp)
set_target_properties(${MOD_LIBRARY_NAME} PROPERTIES
    OUTPUT_NAME "${MOD_LIBRARY_NAME}"
    BUILD_RPATH ""
    INSTALL_RPATH "")
target_include_directories(${MOD_LIBRARY_NAME} PRIVATE
    src
    "${LEVI_PRELOADER_INCLUDE_DIR}")
target_compile_options(${MOD_LIBRARY_NAME} PRIVATE
    $<$<COMPILE_LANG_AND_ID:CXX,Clang,GNU>:-Wall -Wextra -Wpedantic -fvisibility=hidden -fvisibility-inlines-hidden>)
target_link_options(${MOD_LIBRARY_NAME} PRIVATE "-Wl,-z,max-page-size=16384")
find_library(log-lib log REQUIRED)
target_link_libraries(${MOD_LIBRARY_NAME} PRIVATE
    ${log-lib}
    nlohmann_json::nlohmann_json
    Boost::pfr
    magic_enum::magic_enum
    fmt::fmt)
if(TARGET preloader)
    target_link_libraries(${MOD_LIBRARY_NAME} PRIVATE preloader)
else()
    message(FATAL_ERROR "The public preloader SDK target was not created. Refusing to build an unresolved-symbol library.")
endif()

if(NOT CMAKE_STRIP)
    message(FATAL_ERROR "CMAKE_STRIP is required to strip the generated library.")
endif()
add_custom_command(TARGET ${MOD_LIBRARY_NAME} POST_BUILD
    COMMAND "${CMAKE_STRIP}" --strip-unneeded "$<TARGET_FILE:${MOD_LIBRARY_NAME}>"
    COMMENT "Stripping symbols from $<TARGET_FILE_NAME:${MOD_LIBRARY_NAME}>"
    VERBATIM)

configure_file("${CMAKE_CURRENT_SOURCE_DIR}/manifest.json.in"
               "${CMAKE_CURRENT_BINARY_DIR}/manifest.json" @ONLY)
set(LEVI_PACKAGE_ROOT "${CMAKE_CURRENT_BINARY_DIR}/package")
set(LEVI_PACKAGE_MOD_DIR "${LEVI_PACKAGE_ROOT}/${MOD_ID}")
set(LEVI_PACKAGE_FILE "${CMAKE_CURRENT_BINARY_DIR}/${MOD_ID}-${MOD_VERSION}-${CMAKE_ANDROID_ARCH_ABI}.levipack")
add_custom_target(levi_package
    COMMAND "${CMAKE_COMMAND}" -E rm -rf "${LEVI_PACKAGE_ROOT}"
    COMMAND "${CMAKE_COMMAND}" -E make_directory "${LEVI_PACKAGE_MOD_DIR}"
    COMMAND "${CMAKE_COMMAND}" -E copy "$<TARGET_FILE:${MOD_LIBRARY_NAME}>" "${LEVI_PACKAGE_MOD_DIR}/lib${MOD_LIBRARY_NAME}.so"
    COMMAND "${CMAKE_COMMAND}" -E copy "${CMAKE_CURRENT_BINARY_DIR}/manifest.json" "${LEVI_PACKAGE_MOD_DIR}/manifest.json"
    COMMAND "${CMAKE_COMMAND}" -E copy_directory "${CMAKE_CURRENT_SOURCE_DIR}/config" "${LEVI_PACKAGE_MOD_DIR}/config"
    COMMAND "${CMAKE_COMMAND}" -E chdir "${LEVI_PACKAGE_ROOT}" "${CMAKE_COMMAND}" -E tar "cf" "${LEVI_PACKAGE_FILE}" --format=zip "${MOD_ID}"
    WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
    DEPENDS ${MOD_LIBRARY_NAME}
    COMMENT "Packaging ${MOD_ID} for LeviLauncher"
    VERBATIM)

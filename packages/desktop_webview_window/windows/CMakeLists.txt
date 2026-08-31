cmake_minimum_required(VERSION 3.14)
set(PROJECT_NAME "desktop_webview_window")
project(${PROJECT_NAME} LANGUAGES CXX)

# This value is used when generating builds using this plugin, so it must
# not be changed
set(PLUGIN_NAME "desktop_webview_window_plugin")

add_library(${PLUGIN_NAME} SHARED
        "desktop_webview_window_plugin.cpp"
        "webview_window.h"
        "webview_window.cc"
        "strconv.h"
        "utils.h"
        "utils.cc"
        "flutter_view.h"
        "flutter_view.cc"
        "web_view.h"
        "web_view.cc"
        "message_channel_plugin.h"
        "message_channel_plugin.cc"
        "web_view_window_plugin.h"
        "web_view_window_plugin.cc"
        )

apply_standard_settings(${PLUGIN_NAME})
set_target_properties(${PLUGIN_NAME} PROPERTIES
        CXX_VISIBILITY_PRESET hidden)
target_compile_definitions(${PLUGIN_NAME} PRIVATE FLUTTER_PLUGIN_IMPL)
target_include_directories(${PLUGIN_NAME} INTERFACE
        "${CMAKE_CURRENT_SOURCE_DIR}/include")
target_link_libraries(${PLUGIN_NAME} PRIVATE flutter flutter_wrapper_plugin flutter_wrapper_app)

# WebView2Loader is architecture-specific. The published 0.3.0 package hardcodes
# the x64 loader, which makes native Windows ARM64 builds fail with LNK4272.
# Keep the SDK version aligned with the WebView2 headers vendored by this plugin
# and fetch the matching loader from Microsoft's pinned NuGet package.
if(FLUTTER_TARGET_PLATFORM STREQUAL "windows-arm64")
        set(WEBVIEW2_ARCH "arm64")
elseif(FLUTTER_TARGET_PLATFORM STREQUAL "windows-x64")
        set(WEBVIEW2_ARCH "x64")
else()
        message(FATAL_ERROR "Unsupported Windows target for desktop_webview_window: ${FLUTTER_TARGET_PLATFORM}")
endif()

set(WEBVIEW2_VERSION "1.0.992.28")
set(WEBVIEW2_PACKAGE_DIR "${CMAKE_CURRENT_BINARY_DIR}/webview2-${WEBVIEW2_VERSION}")
set(WEBVIEW2_NUPKG "${CMAKE_CURRENT_BINARY_DIR}/Microsoft.Web.WebView2.${WEBVIEW2_VERSION}.nupkg")
set(WEBVIEW2_DLL "${WEBVIEW2_PACKAGE_DIR}/build/native/${WEBVIEW2_ARCH}/WebView2Loader.dll")
set(WEBVIEW2_IMPLIB "${WEBVIEW2_PACKAGE_DIR}/build/native/${WEBVIEW2_ARCH}/WebView2Loader.dll.lib")

if(NOT EXISTS "${WEBVIEW2_DLL}" OR NOT EXISTS "${WEBVIEW2_IMPLIB}")
        file(DOWNLOAD
                "https://www.nuget.org/api/v2/package/Microsoft.Web.WebView2/${WEBVIEW2_VERSION}"
                "${WEBVIEW2_NUPKG}"
                STATUS WEBVIEW2_DOWNLOAD_STATUS
                TLS_VERIFY ON
                SHOW_PROGRESS)
        list(GET WEBVIEW2_DOWNLOAD_STATUS 0 WEBVIEW2_DOWNLOAD_CODE)
        list(GET WEBVIEW2_DOWNLOAD_STATUS 1 WEBVIEW2_DOWNLOAD_MESSAGE)
        if(NOT WEBVIEW2_DOWNLOAD_CODE EQUAL 0)
                message(FATAL_ERROR "Failed to download Microsoft.Web.WebView2 ${WEBVIEW2_VERSION}: ${WEBVIEW2_DOWNLOAD_MESSAGE}")
        endif()

        file(MAKE_DIRECTORY "${WEBVIEW2_PACKAGE_DIR}")
        execute_process(
                COMMAND "${CMAKE_COMMAND}" -E tar xvf "${WEBVIEW2_NUPKG}"
                WORKING_DIRECTORY "${WEBVIEW2_PACKAGE_DIR}"
                RESULT_VARIABLE WEBVIEW2_EXTRACT_RESULT)
        if(NOT WEBVIEW2_EXTRACT_RESULT EQUAL 0)
                message(FATAL_ERROR "Failed to extract Microsoft.Web.WebView2 ${WEBVIEW2_VERSION}")
        endif()
endif()

if(NOT EXISTS "${WEBVIEW2_DLL}")
        message(FATAL_ERROR "WebView2Loader.dll was not found for ${WEBVIEW2_ARCH}: ${WEBVIEW2_DLL}")
endif()
if(NOT EXISTS "${WEBVIEW2_IMPLIB}")
        message(FATAL_ERROR "WebView2Loader import library was not found for ${WEBVIEW2_ARCH}: ${WEBVIEW2_IMPLIB}")
endif()

add_library(Webview2 SHARED IMPORTED GLOBAL)
set_property(TARGET Webview2 PROPERTY IMPORTED_LOCATION "${WEBVIEW2_DLL}")
set_property(TARGET Webview2 PROPERTY IMPORTED_IMPLIB "${WEBVIEW2_IMPLIB}")

target_link_libraries(${PLUGIN_NAME} PRIVATE Webview2 uxtheme)

# List of absolute paths to libraries that should be bundled with the plugin
set(desktop_webview_window_bundled_libraries
        "$<TARGET_FILE:Webview2>"
        PARENT_SCOPE
        )

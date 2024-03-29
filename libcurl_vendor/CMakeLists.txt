cmake_minimum_required(VERSION 3.5)

project(libcurl_vendor)

find_package(ament_cmake REQUIRED)

set(PACKAGE_VERSION "1.0.0")

macro(build_libcurl)
  set(extra_cxx_flags)
  set(extra_c_flags)
  if(NOT WIN32)
    list(APPEND extra_cxx_flags "-std=c++14 -w")
    list(APPEND extra_c_flags "-w")
  endif()

  set(PKGCONFIG_FOR_OPENSSL)
  if(APPLE)
    set(PKGCONFIG_FOR_OPENSSL "/usr/local/opt/openssl/lib/pkgconfig")
  endif()

  if(WIN32 AND NOT ${CMAKE_VERBOSE_MAKEFILE})
    set(should_log ON)  # prevent warnings in Windows CI
  else()
    set(should_log OFF)
  endif()

  include(ExternalProject)
  if(WIN32)
    ExternalProject_Add(curl-7.57.0
      URL https://github.com/curl/curl/releases/download/curl-7_57_0/curl-7.57.0.tar.gz
      URL_MD5 c7aab73aaf5e883ca1d7518f93649dc2
      LOG_CONFIGURE ${should_log}
      LOG_BUILD ${should_log}
      CMAKE_ARGS
        -DENABLE_MANUAL=OFF
        -DBUILD_TESTING=OFF
        -DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_BINARY_DIR}/libcurl_install
        -Wno-dev
      TIMEOUT 600
    )
  else()
    ExternalProject_Add(curl-7.57.0
      URL https://github.com/curl/curl/releases/download/curl-7_57_0/curl-7.57.0.tar.gz
      URL_MD5 c7aab73aaf5e883ca1d7518f93649dc2
      CONFIGURE_COMMAND
        <SOURCE_DIR>/configure
        CFLAGS=${extra_c_flags}
        CXXFLAGS=${extra_cxx_flags}
        PKG_CONFIG_PATH=$PKG_CONFIG_PATH:${PKGCONFIG_FOR_OPENSSL}
        --prefix=${CMAKE_CURRENT_BINARY_DIR}/libcurl_install
        --with-ssl
      BUILD_COMMAND ${MAKE}
      INSTALL_COMMAND make install
      TIMEOUT 600
    )
  endif()

  # The external project will install to the build folder, but we'll install that on make install.
  install(
    DIRECTORY
      ${CMAKE_CURRENT_BINARY_DIR}/libcurl_install/
    DESTINATION
      ${CMAKE_INSTALL_PREFIX}/opt/libcurl_vendor
  )
endmacro()

find_package(CURL QUIET)

if(NOT CURL_FOUND)
  # System curl not found, build one locally.
  build_libcurl()

  if(WIN32)
    ament_environment_hooks(env_hook/libcurl_vendor_library_path.bat)
  else()
    ament_environment_hooks(env_hook/libcurl_vendor_library_path.sh)
  endif()
endif()

ament_package(
  CONFIG_EXTRAS "libcurl_vendor-extras.cmake.in"
)

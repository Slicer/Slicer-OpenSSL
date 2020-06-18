cmake_minimum_required(VERSION 3.13.4)

#
# This project allows to configure, build, archive and upload an OpenSSL install tree.
#

if(DEFINED UPLOAD_PACKAGE_SCRIPT)
  message("----------------------")
  message("${PACKAGE_NAME}")
  message("  os: ${PACKAGE_OS}")
  message("  arch: ${PACKAGE_ARCH}")
  message("  version: ${PACKAGE_VERSION}")

  set(gh_project "Slicer/Slicer-OpenSSL")
  set(ENV{GITHUB_TOKEN} ${GITHUB_TOKEN})

  execute_process(
    COMMAND ${GITHUB_RELEASE_EXECUTABLE} release ${gh_project} create ${PACKAGE_VERSION} --publish
    )

  execute_process(
    COMMAND ${GITHUB_RELEASE_EXECUTABLE} asset ${gh_project} upload ${PACKAGE_VERSION} ${PACKAGE_NAME}
    WORKING_DIRECTORY ${ARCHIVE_DIR}
    )
  return()
endif()

project(OpenSSL)
include(ExternalProject)

set(OPENSSL_VERSION "1.1.1g" CACHE STRING "Verison of OpenSSL to build")

set(source_archive_1.0.1h_hash "MD5=9e380f6f9cc497bd2f47c944019239ac")
set(source_archive_1.0.2n_hash "MD5=2d1a1dcef6169e3ab8fd30b815c3e827")
set(source_archive_1.1.1g_hash "SHA256=281e4f13142b53657bd154481e18195b2d477572fdffa8ed1065f73ef5a19777")

set(version ${OPENSSL_VERSION})
if(NOT DEFINED source_archive_${version}_hash)
  message(FATAL_ERROR "OpenSSL version ${version} is not recognized by ${PROJECT_NAME} build system: "
                      "Variable 'source_archive_${version}_hash' is not defined !")
endif()
string(REPLACE "." "_" version_underscore ${version})
set(source_archive_name "OpenSSL_${version_underscore}")
math(EXPR bitness "${CMAKE_SIZEOF_VOID_P} * 8")
set(binary_archive_name "${source_archive_name}-install-msvc${MSVC_VERSION}-${bitness}")
set(binary_archive_name_with_config "${binary_archive_name}-${CMAKE_BUILD_TYPE}")

find_program(PERL_COMMAND "Path to perl executable")
if(NOT PERL_COMMAND)
  message(FATAL_ERROR "PERL_COMMAND is not set !")
endif()
get_filename_component(PERL_BIN_DIR ${PERL_COMMAND} PATH)

option(UPLOAD_PACKAGES "Upload package(s)" ON)
if(UPLOAD_PACKAGES)
  find_program(GITHUB_RELEASE_EXECUTABLE "Path to GitHub release executable")
  if(NOT GITHUB_RELEASE_EXECUTABLE)
    message(FATAL_ERROR "GITHUB_RELEASE_EXECUTABLE is not set !")
  endif()
  set(GITHUB_TOKEN "" CACHE STRING "GitHub token")
  if("x${GITHUB_TOKEN}" STREQUAL "x")
    message(FATAL_ERROR "GITHUB_TOKEN is not set !")
  endif()
  set(package_upload_options
    -DGITHUB_RELEASE_EXECUTABLE:STRING=${GITHUB_RELEASE_EXECUTABLE}
    -DGITHUB_TOKEN:STRING=${GITHUB_TOKEN}
    )
endif()

option(SUPERBUILD "Build ${proj} Debug and Release tree" ON)

set(proj ${PROJECT_NAME})

if(NOT SUPERBUILD)

  set(do_script "")

  if(CMAKE_SIZEOF_VOID_P EQUAL 4)
    set(config "VC-WIN32")
    if(OPENSSL_VERSION VERSION_LESS "1.1.0")
      set(do_script "call ms\\do_ms.bat")
    endif()
  elseif(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(config "VC-WIN64A")
    if(OPENSSL_VERSION VERSION_LESS "1.1.0")
      set(do_script "call ms\\do_win64a.bat")
    endif()
  endif()
  if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(config "debug-${config}")
  endif()

  file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/${proj}-configure.bat"
  "set PATH=%PATH%;${PERL_BIN_DIR}
perl Configure ${config} no-asm --prefix=${CMAKE_BINARY_DIR}/${binary_archive_name_with_config} --openssldir=${CMAKE_BINARY_DIR}/${binary_archive_name_with_config}
${do_script}
")

  set(nmake_options "")
  if(OPENSSL_VERSION VERSION_LESS "1.1.0")
    set(nmake_options "-f ms\\ntdll.mak")
  endif()

  file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/${proj}-build.bat"
  "set PATH=%PATH%;${PERL_BIN_DIR}
nmake ${nmake_options}")

  file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/${proj}-install.bat"
  "set PATH=%PATH%;${PERL_BIN_DIR}
nmake ${nmake_options} install")

  ExternalProject_Add(${proj}
    # Use github archive instead of the one hosted on openssl.org because of CMake bug #13251
    URL "https://github.com/openssl/openssl/archive/${source_archive_name}.tar.gz"
    URL_HASH "${source_archive_${version}_hash}"
    DOWNLOAD_DIR ${CMAKE_CURRENT_BINARY_DIR}
    SOURCE_DIR ${CMAKE_BINARY_DIR}/${proj}
    BUILD_IN_SOURCE 1
    CONFIGURE_COMMAND "${CMAKE_CURRENT_BINARY_DIR}/${proj}-configure.bat"
    BUILD_COMMAND "${CMAKE_CURRENT_BINARY_DIR}/${proj}-build.bat"
    INSTALL_COMMAND "${CMAKE_CURRENT_BINARY_DIR}/${proj}-install.bat"
    )

  ExternalProject_Add_Step(${proj} create-archive
    COMMAND ${CMAKE_COMMAND} -E
      tar czvf ${binary_archive_name_with_config}.tar.gz ${binary_archive_name_with_config}
    DEPENDEES install
    )
  ExternalProject_Add_StepTargets(${proj} create-archive)

  if(UPLOAD_PACKAGES)
    ExternalProject_Add_Step(${proj} upload-archive
      COMMAND ${CMAKE_COMMAND}
        ${package_upload_options}
        -DARCHIVE_DIR:PATH=${CMAKE_BINARY_DIR}
        -DPACKAGE_ARCH:STRING=MSVC${MSVC_VERSION}_${bitness}-bit
        -DPACKAGE_NAME:STRING=${binary_archive_name_with_config}.tar.gz
        -DPACKAGE_OS:STRING=Windows-${CMAKE_BUILD_TYPE}
        -DPACKAGE_VERSION:STRING=${version}
        -DUPLOAD_PACKAGE_SCRIPT:BOOL=ON -P ${CMAKE_CURRENT_LIST_FILE}
      DEPENDEES create-archive
      EXCLUDE_FROM_MAIN 1
      )
    ExternalProject_Add_StepTargets(${proj} upload-archive)
  endif()

  if(DEFINED MULTICONFIG_DIR)
    ExternalProject_Add_Step(${proj} copy-into-multiconfig-dir
      COMMAND ${CMAKE_COMMAND} -E copy_directory
        ${CMAKE_BINARY_DIR}/${binary_archive_name_with_config}
        ${MULTICONFIG_DIR}/${CMAKE_BUILD_TYPE}
      DEPENDEES install
      )
    ExternalProject_Add_StepTargets(${proj} copy-into-multiconfig-dir)
  endif()

else()

  set(multiconfig_dir ${CMAKE_BINARY_DIR}/${binary_archive_name})
  file(MAKE_DIRECTORY ${multiconfig_dir})

  set(ep_cmake_cache_args
    -DPERL_COMMAND:FILEPATH=${PERL_COMMAND}
    -DSUPERBUILD:BOOL=OFF
    -DUPLOAD_PACKAGES:BOOL=${UPLOAD_PACKAGES}
    ${package_upload_options}
    -DMULTICONFIG_DIR:PATH=${multiconfig_dir}
    )

  ExternalProject_Add(${proj}-Release
    DOWNLOAD_COMMAND ""
    SOURCE_DIR "${CMAKE_SOURCE_DIR}"
    BINARY_DIR "${CMAKE_BINARY_DIR}/${proj}-Release"
    CMAKE_CACHE_ARGS ${ep_cmake_cache_args} -DCMAKE_BUILD_TYPE:STRING=Release
    INSTALL_COMMAND ""
    )

  if(UPLOAD_PACKAGES)
    ExternalProject_Add_Step(${proj}-Release upload-archive
      COMMAND ${CMAKE_COMMAND}
        --build "${CMAKE_BINARY_DIR}/${proj}-Release"
        --config Release
        --target OpenSSL-upload-archive
      DEPENDEES build
      )
    ExternalProject_Add_StepTargets(${proj}-Release upload-archive)
  endif()

  ExternalProject_Add(${proj}-Debug
    DOWNLOAD_COMMAND ""
    SOURCE_DIR "${CMAKE_SOURCE_DIR}"
    BINARY_DIR "${CMAKE_BINARY_DIR}/${proj}-Debug"
    CMAKE_CACHE_ARGS ${ep_cmake_cache_args} -DCMAKE_BUILD_TYPE:STRING=Debug
    INSTALL_COMMAND ""
    )

  if(UPLOAD_PACKAGES)
    ExternalProject_Add_Step(${proj}-Debug upload-archive
      COMMAND ${CMAKE_COMMAND}
        --build "${CMAKE_BINARY_DIR}/${proj}-Debug"
        --config Debug
        --target OpenSSL-upload-archive
      DEPENDEES build
      )
    ExternalProject_Add_StepTargets(${proj}-Debug upload-archive)
  endif()

  ExternalProject_Add(${proj}-MultiConfig
    DOWNLOAD_COMMAND ""
    CONFIGURE_COMMAND ""
    BUILD_COMMAND
      ${CMAKE_COMMAND} -E chdir ${CMAKE_BINARY_DIR}
      ${CMAKE_COMMAND} -E tar czvf ${binary_archive_name}.tar.gz ${binary_archive_name}
    INSTALL_COMMAND ""
    DEPENDS
      ${proj}-Debug
      ${proj}-Release
    STEP_TARGETS build
    )

  if(UPLOAD_PACKAGES)
    ExternalProject_Add_Step(${proj}-MultiConfig upload-archive
      COMMAND ${CMAKE_COMMAND}
        ${package_upload_options}
        -DARCHIVE_DIR:PATH=${CMAKE_BINARY_DIR}
        -DPACKAGE_ARCH:STRING=MSVC${MSVC_VERSION}_${bitness}-bit
        -DPACKAGE_NAME:STRING=${binary_archive_name}.tar.gz
        -DPACKAGE_OS:STRING=Windows
        -DPACKAGE_VERSION:STRING=${version}
        -DUPLOAD_PACKAGE_SCRIPT:BOOL=ON -P ${CMAKE_CURRENT_LIST_FILE}
      DEPENDEES build
      )
    ExternalProject_Add_StepTargets(${proj}-MultiConfig upload-archive)
  endif()

endif()

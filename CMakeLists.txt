cmake_minimum_required(VERSION 2.8.10.2)

#
# This project allows to configure, build, archive and upload an OpenSSL install tree.
#

if(DEFINED UPLOAD_PACKAGE_SCRIPT)

  set(MIDAS_URL "http://packages.kitware.com")
  set(FOLDER_ID 203)
  set(APPLICATION_ID 20)
  set(PACKAGE_TYPE "Tar.gz Archive")

  set(midas_api_script ${ARCHIVE_DIR}/MidasAPI.cmake)
  if(NOT EXISTS ${midas_api_script})
    file(DOWNLOAD ${MIDAS_URL}/api/rest?method=midas.packages.script.download ${midas_api_script})
  endif()
  include(${midas_api_script})

  message("----------------------")
  message("${PACKAGE_NAME}")
  message("  os: ${PACKAGE_OS}")
  message("  arch: ${PACKAGE_ARCH}")
  message("  version: ${PACKAGE_VERSION}")

  midas_api_package_upload(
    API_URL ${MIDAS_URL}
    API_EMAIL ${MIDAS_USER}
    API_KEY ${MIDAS_API_KEY}
    FILE "${ARCHIVE_DIR}/${PACKAGE_NAME}"
    NAME ${PACKAGE_NAME}
    FOLDER_ID ${FOLDER_ID}
    APPLICATION_ID ${APPLICATION_ID}
    OS ${PACKAGE_OS}
    ARCH ${PACKAGE_ARCH}
    PACKAGE_TYPE ${PACKAGE_TYPE}
    SUBMISSION_TYPE Experimental
    RELEASE "${PACKAGE_VERSION}"
    RESULT_VARNAME upload_status
    )
  message("Upload status: ${upload_status}")
  return()
endif()

project(OpenSSL)
include(ExternalProject)

set(version "1.0.1h")
string(REPLACE "." "_" version_underscore ${version})
set(source_archive_name "OpenSSL_${version_underscore}")
math(EXPR bitness "${CMAKE_SIZEOF_VOID_P} * 8")
set(binary_archive_name "${source_archive_name}-install-msvc${MSVC_VERSION}-${bitness}")
set(binary_archive_name_with_config "${binary_archive_name}-${CMAKE_BUILD_TYPE}")

option(UPLOAD_PACKAGES "Upload package(s)" ON)
option(SUPERBUILD "Build ${proj} Debug and Release tree" ON)

if(UPLOAD_PACKAGES)
  set(MIDAS_USER "" CACHE STRING "Midas User")
  set(MIDAS_API_KEY "" CACHE STRING "Midas API Key")
  set(package_upload_options
    -DMIDAS_USER:STRING=${MIDAS_USER}
    -DMIDAS_API_KEY:STRING=${MIDAS_API_KEY}
    )
endif()

find_program(PERL_COMMAND "Path to perl executable")
if(NOT PERL_COMMAND)
  message(FATAL_ERROR "PERL_COMMAND is not set !")
endif()
get_filename_component(PERL_BIN_DIR ${PERL_COMMAND} PATH)

set(proj ${PROJECT_NAME})

if(NOT SUPERBUILD)

  if(CMAKE_SIZEOF_VOID_P EQUAL 4)
    set(config "VC-WIN32")
    set(do_script "do_ms.bat")
  elseif(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(config "VC-WIN64A")
    set(do_script "do_win64a.bat")
  endif()
  if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(config "debug-${config}")
  endif()

  file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/${proj}-configure.bat"
  "set PATH=%PATH%;${PERL_BIN_DIR}
perl Configure ${config} no-asm --prefix=${CMAKE_BINARY_DIR}/${binary_archive_name_with_config}
call ms\\${do_script}")

  file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/${proj}-build.bat"
  "set PATH=%PATH%;${PERL_BIN_DIR}
nmake -f ms\\ntdll.mak")

  file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/${proj}-install.bat"
  "set PATH=%PATH%;${PERL_BIN_DIR}
nmake -f ms\\ntdll.mak install")

  ExternalProject_Add(${proj}
    # Use github archive instead of the one hosted on openssl.org because of CMake bug #13251
    URL "https://github.com/openssl/openssl/archive/${source_archive_name}.tar.gz"
    URL_MD5 "9e380f6f9cc497bd2f47c944019239ac"
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

  ExternalProject_Add(${proj}-Debug
    DOWNLOAD_COMMAND ""
    SOURCE_DIR "${CMAKE_SOURCE_DIR}"
    BINARY_DIR "${CMAKE_BINARY_DIR}/${proj}-Debug"
    CMAKE_CACHE_ARGS ${ep_cmake_cache_args} -DCMAKE_BUILD_TYPE:STRING=Debug
    INSTALL_COMMAND ""
    )

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

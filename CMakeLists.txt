# Adapted from Izaak Beekman by Lianhua Zhu, Sept. 2019

# the original copyright statments
# CMake Configuration and build added by Izaak Beekman -- May, 2014

#---------------------------------------------------------------------
# Copy right (c) 2019, Lianhua Zhu
# All rights reserved.

# This file is contributed to the LIB_VTK_IO project, and
# is licensed under the terms of LIB_VTK_IO license. The LIB_VTK_IO
# license is located in the LICENSE* files which must be distributed with
# this software. The contributing author, Lianhua Zhu, retains all
# rights permitted by the terms of the LIB_VTK_IO license.
#---------------------------------------------------------------------

cmake_minimum_required ( VERSION 2.8.8 FATAL_ERROR )

# Use MSVS folders to organize projects on windows
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

set(PROJECT_DESCRIPTION "A Fortran VTK File IO API")
set(PROJECT_URL "https://github.com/zhulianhua/Lib_VTK_IO")

# Set the type/configuration of build to perform
set ( CMAKE_CONFIGURATION_TYPES "Debug" "Release" "MinSizeRel" "RelWithDebInfo" )
set ( CMAKE_BUILD_TYPE "Release"
  CACHE STRING "Select which configuration to build." )

enable_language ( Fortran )
include ( "cmake/pickFortranCompilerFlags.cmake" )

set_property ( CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS ${CMAKE_CONFIGURATION_TYPES} )

# Check for in-source builds and error out if found
# Provides an advanced option to allow in source builds
include ( "cmake/checkOutOfSource.cmake" )

#----------------------------------------------
# DEFINE PATHS
#----------------------------------------------
SET(LIB_PATH ${CMAKE_SOURCE_DIR}/src/lib)
SET(TESTS_PATH ${CMAKE_SOURCE_DIR}/src/tests)

#---------------------
# Declare project name
#---------------------
project ( lib_vtk_io NONE )

#---------------------
# Enable test
#---------------------
IF(NOT ${PROJECT_NAME}_ENABLE_TESTS)
    option(${PROJECT_NAME}_ENABLE_TESTS "Enable/disable tests" OFF)
ENDIF(NOT ${PROJECT_NAME}_ENABLE_TESTS)

#----------------------------------
# Get version (semantic versioning)
# C.F. semver.org
#----------------------------------
file ( STRINGS "${CMAKE_SOURCE_DIR}/.VERSION" VERSION )
string( REPLACE "." ";" VERSION_LIST ${VERSION} )
list(GET VERSION_LIST 0 VERSION_MAJOR)
list(GET VERSION_LIST 1 VERSION_MINOR)
list(GET VERSION_LIST 2 VERSION_PATCH)
set(PROJECT_VERSION "${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}")
message ( STATUS "CMake build configuration for lib_vtk_io ${PROJECT_VERSION}" )

#-------------------------------------
# Collect source files for the library
#-------------------------------------
set ( VTK_LIB_SRCS src/lib/IR_Precision.f90
                   src/lib/Lib_Base64.f90
                   src/lib/Lib_Pack_Data.f90
                   src/lib/Lib_VTK_IO.f90)

file ( GLOB VTK_TEST_SRCS "src/tests/*.f90" )

#-----------------------------------------
# Collect all the mod files into their own
# directory to ease installation issues
#-----------------------------------------
set ( MODULE_DIR "${CMAKE_BINARY_DIR}/include" )

#-------------------------------------
# Define where our files get installed
#-------------------------------------
set ( USE_GNU_INSTALL_CONVENTION FALSE
  CACHE BOOL
  "Install library, module file and documentation to standard GNU locations. Do not use this if supporting multiple Fortran compilers" )

# Set the package name to be specific to the compiler used, so that
# versions compiled with different compilers can be installed in parallel
string ( TOLOWER ${CMAKE_PROJECT_NAME}-${CMAKE_Fortran_COMPILER_ID} PACKAGE_NAME )
set ( PACKAGE_VERSION "${PACKAGE_NAME}-${VERSION}" )

if (USE_GNU_INSTALL_CONVENTION)
  include(GNUInstallDirs)
  set ( INSTALL_MOD_DIR "${CMAKE_INSTALL_INCLUDEDIR}" )
  set ( INSTALL_LIB_DIR "${CMAKE_INSTALL_LIBDIR}")
  set( ABS_LIB_INSTALL_DIR "\${CMAKE_INSTALL_FULL_LIBDIR}" )
else ()
  # Most of this could be 'wrong' for Windows/Cygwin
  set ( INSTALL_MOD_DIR    "${PACKAGE_VERSION}/lib" )
  set ( INSTALL_LIB_DIR    "${INSTALL_MOD_DIR}" )
  set( ABS_LIB_INSTALL_DIR "\${CMAKE_INSTALL_PREFIX}/${INSTALL_LIB_DIR}" )
endif ()


# Put package export CMake files where they can be found
# use `find_package ( lib_vtk_io-${CMAKE_Fortran_COMPILER_ID} <version> REQUIRED )`
if (USE_GNU_INSTALL_CONVENTION)
  set ( EXPORT_INSTALL_DIR "${INSTALL_LIB_DIR}/cmake/${PACKAGE_VERSION}" )
else ()
  set ( EXPORT_INSTALL_DIR "${PACKAGE_VERSION}/cmake" )
endif ()

if ( "${CMAKE_SYSTEM_NAME}" MATCHES "Darwin" )
  set ( ENABLE_DYLIBS_USE_RPATH TRUE CACHE BOOL
    "Enable @rpath install name for dylibs" )
  mark_as_advanced ( ENABLE_DYLIBS_USE_RPATH )
endif ()

if ( ENABLE_DYLIBS_USE_RPATH )
  set ( CMAKE_MACOSX_RPATH TRUE )
else ()
  set ( CMAKE_INSTALL_NAME_DIR
    "${ABS_LIB_INSTALL_DIR}" )
endif ()

#---------------------------------------------
# Build a shared and static library by default
#---------------------------------------------
set ( LIB_NAME ${CMAKE_PROJECT_NAME} )
add_library ( ${LIB_NAME}        SHARED ${VTK_LIB_SRCS} )
add_library ( ${LIB_NAME}-static STATIC ${VTK_LIB_SRCS} )

set_target_properties ( ${LIB_NAME}-static
  PROPERTIES
  OUTPUT_NAME ${LIB_NAME}
  if(NOT MSVC_IDE)
  PREFIX lib
  endif()
  VERSION ${VERSION}
  ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib
  Fortran_MODULE_DIRECTORY ${MODULE_DIR} )
set_target_properties ( ${LIB_NAME}
  PROPERTIES
  OUTPUT_NAME ${LIB_NAME}
  if(NOT MSVC_IDE)
  PREFIX lib
  endif()
  SOVERSION ${VERSION_MAJOR}.${VERSION_MINOR}
  VERSION ${VERSION}
  LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib
  Fortran_MODULE_DIRECTORY ${CMAKE_BINARY_DIR} )

#-------------------------------------
# Build the documentation with FORD
#-------------------------------------
set ( SKIP_DOC_GEN TRUE CACHE BOOL
  "Disable building the API documentation with FORD" )
if ( NOT SKIP_DOC_GEN )
  find_program ( FORD ford )
  if ( FORD ) # Found
    file ( COPY "${CMAKE_SOURCE_DIR}/media" DESTINATION "${CMAKE_BINARY_DIR}/" )
    file ( GLOB_RECURSE PAGES_FILES "${CMAKE_SOURCE_DIR}/pages/*.*")
    set ( DOC_DIR "${CMAKE_BINARY_DIR}/doc" )
    set ( PAGES_DIR "${CMAKE_SOURCE_DIR}/pages" )
    set ( PROJ_DIR "${CMAKE_SOURCE_DIR}/src" )
    set ( FORD_PROJECT_FILE "${CMAKE_SOURCE_DIR}/json-fortran.md" )
    # Pick the preprocessor to use based on the Fortran compiler
    if ( "${CMAKE_Fortran_COMPILER_ID}" MATCHES "Intel" )
      set ( FPP "fpp\n" )
    else ()
      set ( FPP "gfortran -E\n" ) # default to gfortran -E for gfortran and unsupported compilers
    endif ()
    file ( WRITE "${CMAKE_BINARY_DIR}/.PREPROCESSOR" "${FPP}" )
    # Dynamically generate the FORD outputs list
    message ( STATUS "Dynamically computing FORD output information..." )
    if ( NOT (DEFINED FORD_OUTPUTS_CACHED) )
      message ( STATUS "Running FORD to dynamically compute documentation outputs, this could take a while..." )
      execute_process ( COMMAND ${CMAKE_COMMAND} -E remove_directory ${DOC_DIR}
	COMMAND ${CMAKE_COMMAND} -E make_directory ${DOC_DIR}
	COMMAND "${FORD}" --debug -d "${PROJ_DIR}" -o "${DOC_DIR}" -p "${PAGE_DIR}" "${FORD_PROJECT_FILE}" OUTPUT_QUIET )
    else ()
      message ( STATUS "Re-using cached FORD outputs, rather than regenerating them" )
    endif ()
    file ( GLOB_RECURSE FORD_OUTPUTS
      "${DOC_DIR}/*.*" )
    file ( GLOB_RECURSE FORD_CLEAN_OUTPUTS
      "${DOC_DIR}/*.*" )
    if ( (DEFINED FORD_OUTPUTS) AND ( NOT ( "${FORD_OUTPUTS}" STREQUAL "" ) ) )
      message ( STATUS "Caching FORD outputs" )
      set ( FORD_OUTPUTS_CACHED "${FORD_OUTPUTS}"
	CACHE STRING "variable containing FORD outputs to prevent rebuilding FORD docs" FORCE )
    endif ()
    message ( STATUS "Done dynamically computing FORD outputs." )

    foreach ( DOC_SRC_FILE ${VTK_LIB_SRCS} ${VTK_TEST_SRCS} ${CMAKE_SOURCE_DIR}/README.md
        ${CMAKE_SOURCE_DIR}/CHANGELOG.md ${CMAKE_SOURCE_DIR}/CONTRIBUTING.md
        ${CMAKE_SOURCE_DIR}/LICENSE.mit.md ${CMAKE_SOURCE_DIR}/LICENSE.gpl3.md 
        ${PAGES_FILES} )
      list ( APPEND FORD_DEPENDS "${DOC_SRC_FILE}" )
    endforeach ()
    add_custom_command ( OUTPUT ${FORD_OUTPUTS_CACHED}
      COMMAND "${FORD}" --debug ${MACRO_FLAG} -d "${PROJ_DIR}" -o "${DOC_DIR}" -p "${CMAKE_SOURCE_DIR}/pages" "${FORD_PROJECT_FILE}"
      MAIN_DEPENDENCY "${FORD_PROJECT_FILE}"
      DEPENDS ${FORD_DEPENDS}
      COMMENT "Building HTML documentation for ${CMAKE_PROJECT_NAME} using FORD" )
    add_custom_target ( documentation ALL
      DEPENDS ${FORD_OUTPUTS_CACHED} )
    set ( INSTALL_API_DOCUMENTATION TRUE
      CACHE BOOL "Install FORD generated documentation?" )
    if ( INSTALL_API_DOCUMENTATION )
      if ( USE_GNU_INSTALL_CONVENTION )
	install ( DIRECTORY "${DOC_DIR}/" DESTINATION "${CMAKE_INSTALL_DOCDIR}" )
      else ()
	install ( DIRECTORY "${DOC_DIR}/" DESTINATION "${CMAKE_INSTALL_PREFIX}/${PACKAGE_VERSION}/doc" )
      endif ()
    endif ()
  else () # Not found
    message ( WARNING
      "FORD not found! Please set the CMake cache variable FORD to point to the installed FORD executable, and reconfigure or disable building the documentation. FORD can be installed from PYPI with `sudo pip install FORD` or from <https://github.com/cmacmackin/ford> If you do not wish to install FORD and build the JSON-Fortran documentation, then please set the CMake cache variable SKIP_DOC_GEN to TRUE." )
  endif ()
endif ()

#--------------------------
# Handle test related stuff
#--------------------------
set ( ENABLE_TESTS FALSE CACHE BOOL
  "Enable the lib_vtk_io tests." )

#---------------------------------------------------------------------
# Add some tests to ensure that the software is performing as expected
#---------------------------------------------------------------------
if ( ENABLE_TESTS )
    message("skip testing!")
endif()

#-------------------------
# Perform the installation
#-------------------------

install ( TARGETS ${LIB_NAME} ${LIB_NAME}-static
  EXPORT ${PACKAGE_NAME}-targets
  LIBRARY DESTINATION "${INSTALL_LIB_DIR}"
  ARCHIVE DESTINATION "${INSTALL_LIB_DIR}" )

# Code to  fix the dylib install name on Mac.
include ( cmake/fixupInstallNameDir.cmake )

set(MOD_DIR_TO_INSTALL "${MODULE_DIR}")
set(MOD_DESTINATION_DIR "${INSTALL_MOD_DIR}")
install(
  CODE "file(GLOB_RECURSE MODULE_FILES \"${MOD_DIR_TO_INSTALL}/*.mod\")"
  CODE "file(GLOB_RECURSE SUBMOD_FILES \"${MOD_DIR_TO_INSTALL}/*.smod\")"
  CODE "file(INSTALL \${MODULE_FILES} DESTINATION \"\${CMAKE_INSTALL_PREFIX}/${MOD_DESTINATION_DIR}\")"
  CODE "file(INSTALL \${SUBMOD_FILES} DESTINATION \"\${CMAKE_INSTALL_PREFIX}/${MOD_DESTINATION_DIR}\")"
  )

#------------------------------------------
# Add portable unistall command to makefile
#------------------------------------------
# Adapted from the CMake Wiki FAQ
configure_file ( "${CMAKE_SOURCE_DIR}/cmake/uninstall.cmake.in" "${CMAKE_BINARY_DIR}/uninstall.cmake"
    @ONLY)

add_custom_target ( uninstall
    COMMAND ${CMAKE_COMMAND} -P "${CMAKE_BINARY_DIR}/uninstall.cmake" )

#-----------------------------------------------------
# Publicize installed location to other CMake projects
#-----------------------------------------------------
install ( EXPORT ${PACKAGE_NAME}-targets DESTINATION "${EXPORT_INSTALL_DIR}" )

include ( CMakePackageConfigHelpers ) # Standard CMake module
write_basic_package_version_file( "${CMAKE_BINARY_DIR}/${PACKAGE_NAME}-config-version.cmake"
  VERSION ${VERSION}
  COMPATIBILITY SameMajorVersion )

# provides COMPILER_CONSISTENCY_CHECK
include ( cmake/FCompilerConsistencyCheck.cmake )

# install package config file
configure_package_config_file (
  "${CMAKE_SOURCE_DIR}/cmake/pkg/${CMAKE_PROJECT_NAME}-config.cmake.in"
  "${CMAKE_BINARY_DIR}/pkg/${PACKAGE_NAME}-config.cmake"
  INSTALL_DESTINATION "${EXPORT_INSTALL_DIR}"
  PATH_VARS EXPORT_INSTALL_DIR INSTALL_MOD_DIR )

# Install the config and version files so that we can find this project with others
install ( FILES
  "${CMAKE_BINARY_DIR}/pkg/${PACKAGE_NAME}-config.cmake"
  "${CMAKE_BINARY_DIR}/${PACKAGE_NAME}-config-version.cmake"
  DESTINATION "${EXPORT_INSTALL_DIR}" )

#----------------------------------------------
# Make build tree targets accessible for import
#----------------------------------------------
export ( TARGETS ${LIB_NAME} ${LIB_NAME}-static FILE ${PACKAGE_NAME}-targets.cmake )

# build tree package config file, NOT installed
configure_file (
  "${CMAKE_SOURCE_DIR}/cmake/${CMAKE_PROJECT_NAME}-config.cmake.in"
  "${CMAKE_BINARY_DIR}/${PACKAGE_NAME}-config.cmake"
  @ONLY )

set ( ENABLE_BUILD_TREE_EXPORT FALSE CACHE BOOL
  "Add the ${PACKAGE_NAME} build tree to the CMake package registry?" )
if ( ENABLE_BUILD_TREE_EXPORT )
  export ( PACKAGE ${PACKAGE_NAME} )
endif ()

# pkg-config stuff
configure_file(
   "${CMAKE_CURRENT_SOURCE_DIR}/lib_vtk_io.pc.cmake.in"
   "${CMAKE_CURRENT_BINARY_DIR}/lib_vtk_io.pc"
   @ONLY
)
install(FILES
   "${CMAKE_CURRENT_BINARY_DIR}/lib_vtk_io.pc"
   DESTINATION "${INSTALL_LIB_DIR}/pkgconfig"
)

if(MSVC_IDE)
   INCLUDE_DIRECTORIES("src")
   SET(CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -fpp")
endif()

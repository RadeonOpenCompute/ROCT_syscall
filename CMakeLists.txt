################################################################################
##
## Copyright (c) 2016 Advanced Micro Devices, Inc. All rights reserved.
##
## MIT LICENSE:
## Permission is hereby granted, free of charge, to any person obtaining a copy of
## this software and associated documentation files (the "Software"), to deal in
## the Software without restriction, including without limitation the rights to
## use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies
## of the Software, and to permit persons to whom the Software is furnished to do
## so, subject to the following conditions:
##
## The above copyright notice and this permission notice shall be included in all
## copies or substantial portions of the Software.
##
## THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
## IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
## FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.  IN NO EVENT SHALL THE
## AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
## LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
## OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
## SOFTWARE.
##
################################################################################

cmake_minimum_required ( VERSION 3.5.0 )

set ( HSAKMT_NAME "hsakmt" )
set ( HSAKMT_COMPONENT "lib${HSAKMT_NAME}" )
set ( HSAKMT_TARGET "${HSAKMT_NAME}" )

project ( ${HSAKMT_TARGET} )

list ( APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake_modules" )

include ( utils )

## Get the package version. The defaults to 1.0.0.
get_version ( "1.0.0" )

set ( BUILD_VERSION_MAJOR ${VERSION_MAJOR} )
set ( BUILD_VERSION_MINOR ${VERSION_MINOR} )
set ( BUILD_VERSION_PATCH ${VERSION_PATCH} )
set ( LIB_VERSION_STRING "${BUILD_VERSION_MAJOR}.${BUILD_VERSION_MINOR}.${BUILD_VERSION_PATCH}" )
if ( DEFINED VERSION_BUILD AND NOT ${VERSION_BUILD} STREQUAL "" )
    message ( "VERSION BUILD DEFINED ${VERSION_BUILD}" )
    set ( BUILD_VERSION_PATCH "${BUILD_VERSION_PATCH}-${VERSION_BUILD}" )
endif ()
set ( BUILD_VERSION_STRING "${BUILD_VERSION_MAJOR}.${BUILD_VERSION_MINOR}.${BUILD_VERSION_PATCH}" )

## Verbose output.
set ( CMAKE_VERBOSE_MAKEFILE on )

## Compiler flags
set ( CMAKE_C_FLAGS "-fPIC -W -Wall -Wextra -Wno-unused-parameter -Wformat-security -Wswitch-default -Wundef -Wshadow -Wpointer-arith -Wbad-function-cast -Wcast-qual -Wlogical-op -Wstrict-prototypes -Wmissing-prototypes -Wmissing-declarations -Wredundant-decls -Wunreachable-code -std=gnu99 -fvisibility=hidden -O2" )

set ( HSAKMT_LINKER_SCRIPT "${CMAKE_CURRENT_SOURCE_DIR}/src/libhsakmt.ver" )

## Linker Flags
set ( CMAKE_SHARED_LINKER_FLAGS "-Wl,--version-script=${HSAKMT_LINKER_SCRIPT} -Wl,-soname=${HSAKMT_COMPONENT}.so.$(VERSION_MAJOR) -Wl,-z,nodelete -Wl,-no-undefined" )

## Source files
set ( HSAKMT_SRC ${HSAKMT_SRC} src/debug.c )
set ( HSAKMT_SRC ${HSAKMT_SRC} src/events.c )
set ( HSAKMT_SRC ${HSAKMT_SRC} src/fmm.c )
set ( HSAKMT_SRC ${HSAKMT_SRC} src/globals.c )
set ( HSAKMT_SRC ${HSAKMT_SRC} src/libhsakmt.c )
set ( HSAKMT_SRC ${HSAKMT_SRC} src/memory.c )
set ( HSAKMT_SRC ${HSAKMT_SRC} src/openclose.c )
set ( HSAKMT_SRC ${HSAKMT_SRC} src/perfctr.c )
set ( HSAKMT_SRC ${HSAKMT_SRC} src/pmc_table.c )
set ( HSAKMT_SRC ${HSAKMT_SRC} src/queues.c )
set ( HSAKMT_SRC ${HSAKMT_SRC} src/time.c )
set ( HSAKMT_SRC ${HSAKMT_SRC} src/topology.c )
set ( HSAKMT_SRC ${HSAKMT_SRC} src/version.c )

## Include paths
include_directories ( ${CMAKE_CURRENT_SOURCE_DIR}/include )
include_directories ( ${CMAKE_CURRENT_SOURCE_DIR}/src )

## Add the library
add_library ( ${HSAKMT_TARGET} SHARED ${HSAKMT_SRC} )

## Set the VERSION and SOVERSION values
set_property ( TARGET ${HSAKMT_TARGET} PROPERTY VERSION "${LIB_VERSION_STRING}" )
set_property ( TARGET ${HSAKMT_TARGET} PROPERTY SOVERSION "${BUILD_VERSION_MAJOR}" )

find_package(PkgConfig)
pkg_check_modules(PC_LIBPCI REQUIRED libpci)
include_directories ( ${PC_LIBPCI_INCLUDEDIR} )
target_link_libraries ( ${HSAKMT_TARGET}
  pthread rt ${PC_LIBPCI_LIBRARIES}
)

## If the library is a release, strip the target library
if ( "${CMAKE_BUILD_TYPE}" STREQUAL Release )
    add_custom_command ( TARGET ${HSAKMT_TARGET} POST_BUILD COMMAND ${CMAKE_STRIP} ${HSAKMT_COMPONENT}.so )
endif ()

## Create symlinks for packaging and install
add_custom_target ( ${HSAKMT_COMPONENT} ALL WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR} COMMAND ${CMAKE_COMMAND} -E create_symlink ../${HSAKMT_COMPONENT}/include/${HSAKMT_COMPONENT} ${HSAKMT_COMPONENT}-link )
add_custom_target ( ${HSAKMT_COMPONENT}.so-link ALL WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR} COMMAND ${CMAKE_COMMAND} -E create_symlink ../${HSAKMT_COMPONENT}/lib/${HSAKMT_COMPONENT}.so ${HSAKMT_COMPONENT}.so-link )

## Set the install targets
install ( TARGETS ${HSAKMT_TARGET} LIBRARY DESTINATION libhsakmt/lib COMPONENT ${HSAKMT_COMPONENT} )
install ( FILES ${CMAKE_CURRENT_SOURCE_DIR}/LICENSE.md DESTINATION libhsakmt )
install ( FILES ${CMAKE_CURRENT_SOURCE_DIR}/include/hsakmt.h DESTINATION libhsakmt/include/libhsakmt )
install ( FILES ${CMAKE_CURRENT_SOURCE_DIR}/include/hsakmttypes.h DESTINATION libhsakmt/include/libhsakmt )
install ( FILES ${CMAKE_CURRENT_SOURCE_DIR}/include/linux/kfd_ioctl.h DESTINATION libhsakmt/include/libhsakmt/linux )
install ( FILES ${CMAKE_CURRENT_SOURCE_DIR}/include/linux/kfd_sc.h DESTINATION libhsakmt/include/libhsakmt/linux )
install ( FILES ${CMAKE_CURRENT_BINARY_DIR}/${HSAKMT_COMPONENT}-link DESTINATION include RENAME ${HSAKMT_COMPONENT} )
install ( FILES ${CMAKE_CURRENT_BINARY_DIR}/${HSAKMT_COMPONENT}.so-link DESTINATION lib RENAME ${HSAKMT_COMPONENT}.so )

## Packaging directives
set ( CPACK_PACKAGE_NAME "hsakmt-roct-dev" )
set ( CPACK_PACKAGE_VENDOR "AMD" )
set ( CPACK_PACKAGE_VERSION_MAJOR ${BUILD_VERSION_MAJOR} )
set ( CPACK_PACKAGE_VERSION_MINOR ${BUILD_VERSION_MINOR} )
set ( CPACK_PACKAGE_VERSION_PATCH ${BUILD_VERSION_PATCH} )
set ( CPACK_PACKAGE_CONTACT "Advanced Micro Devices Inc." )
set ( CPACK_PACKAGE_DESCRIPTION_SUMMARY "HSAKMT library for AMD KFD support" )
set ( CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE.md" )

# Debian package specific variables
set ( CPACK_DEBIAN_PACKAGE_HOMEPAGE "https://github.com/RadeonOpenCompute/ROCT-Thunk-Interface" )
set ( CPACK_DEBIAN_PACKAGE_CONTROL_EXTRA "${CMAKE_CURRENT_SOURCE_DIR}/DEBIAN/postinst;${CMAKE_CURRENT_SOURCE_DIR}/DEBIAN/prerm" )
set ( CPACK_DEBIAN_PACKAGE_DEPENDS "libpci3" )

# RPM package specific variables
set ( CPACK_RPM_PRE_INSTALL_SCRIPT_FILE "${CMAKE_CURRENT_SOURCE_DIR}/RPM/rpm_post" )
set ( CPACK_RPM_POST_UNINSTALL_SCRIPT_FILE "${CMAKE_CURRENT_SOURCE_DIR}/RPM/rpm_postun" )

include ( CPack )

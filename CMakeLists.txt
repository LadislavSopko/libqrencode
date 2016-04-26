
## ==============================================================================
##
##  CMake project settings
##
## ==============================================================================

## Name of the project handled by CMake
project (QRencode C)

## Minimum required version of CMake to configure the project
cmake_minimum_required (VERSION 2.8)

## Enforced CMake policy
cmake_policy (VERSION 2.8)

## Project version number

set (MAJOR_VERSION "3")
set (MINOR_VERSION "9")
set (MICRO_VERSION "0")
set (PROJECT_VERSION "${MAJOR_VERSION}.${MINOR_VERSION}.${MICRO_VERSION}")

## ==============================================================================
##
##  Configuration options
##
## ==============================================================================

option (WITH_TESTS "Build test programs?"  YES )
option (WITH_TOOLS "Build utility tools?"  YES )



set (ZLIB_ROOT "C:/TRERDP_vc14/xw-3rdp")
set (RDP_ROOT "C:/TRERDP_vc14/xw-3rdp")
#set (PNG_LIBRARY "C:/TRERDP_vc14/xw-3rdp/lib/libpng16.lib")
#set (PNG_PNG_INCLUDE_DIR, "C:/TRERDP_vc14/xw-3rdp/include/libpng16")
include( cmake/FindPNG.cmake )
link_directories("${RDP_ROOT}/lib")

## ==============================================================================
##
##  System inspection
##
## ==============================================================================

## Standard header files

find_package (Doxygen)
#find_package (PNG)
find_package (Threads)

## Check for system include files

include (CheckIncludeFile)

check_include_file (HAVE_DLFCN_H     dlfcn.h    )
check_include_file (HAVE_INTTYPES_H  inttypes.h )
check_include_file (HAVE_MEMORY_H    memory.h   )
check_include_file (HAVE_STDINT_H    stdint.h   )
check_include_file (HAVE_STDLIB_H    stdlib.h   )
check_include_file (HAVE_STRINGS_H   strings.h  )
check_include_file (HAVE_STRING_H    string.h   )

## Iconv library

  if (NOT ICONV_ROOT_DIR)
    ## We need some extra instruction here, in case inspection is done on
    ## Mac OS X: apparently the default /usr/lib/libiconv is missing some of
    ## the symbols, hence we prefer using the library version as installed
    ## via MacPorts.
    if (APPLE)
      set (ICONV_ROOT_DIR /opt/local)
    else (APPLE)
      set (ICONV_ROOT_DIR ${CMAKE_INSTALL_PREFIX})
    endif (APPLE)
  endif (NOT ICONV_ROOT_DIR)

find_path (ICONV_INCLUDES iconv.h
  HINTS ${ICONV_ROOT_DIR} ${CMAKE_INSTALL_PREFIX}
  PATH_SUFFIXES include include/iconv
  )

find_library (ICONV_LIBRARIES iconv
  HINTS ${ICONV_ROOT_DIR} ${CMAKE_INSTALL_PREFIX}
  PATH_SUFFIXES lib
  )

## Memory checker

find_program (MEMORYCHECK_COMMAND
    NAMES purify valgrind boundscheck
    PATH_SUFFIXES bin
    )

## ==============================================================================
##
##  Build instructions
##
## ==============================================================================

include_directories (
  ${PROJECT_SOURCE_DIR}
  ${PROJECT_BINARY_DIR}
  )

if (PNG_INCLUDE_DIR)
  include_directories (${PNG_INCLUDE_DIR})
endif (PNG_INCLUDE_DIR)

if (ICONV_INCLUDES)
  include_directories (${ICONV_INCLUDES})
endif (ICONV_INCLUDES)

##____________________________________________________________________
## Propagation of configuration settings

find_file (HAVE_CONFIGURE_H_IN 
  NAMES configure.h.in config.h.in
  PATHS ${PROJECT_SOURCE_DIR}
  )

if (HAVE_CONFIGURE_H_IN)
  
  ## Process the input file ...
  configure_file (
    ${HAVE_CONFIGURE_H_IN}
    ${PROJECT_SOURCE_DIR}/config.h
    )

  ## ... and register it to the compiler
  add_definitions (-DHAVE_CONFIG_H)

  ## Add generated header file to 'clean' target
  set_property (DIRECTORY
    APPEND PROPERTY ADDITIONAL_MAKE_CLEAN_FILES
    ${PROJECT_SOURCE_DIR}/config.h
    )
  
else (HAVE_CONFIGURE_H_IN)

  add_definitions (-DMAJOR_VERSION=${MAJOR_VERSION})
  add_definitions (-DMINOR_VERSION=${MINOR_VERSION})
  add_definitions (-DMICRO_VERSION=${MICRO_VERSION})
  add_definitions (-DVERSION="${PROJECT_VERSION}")
  
  if (WITH_TESTS)
    add_definitions (-DWITH_TESTS)
  endif (WITH_TESTS)
  
endif (HAVE_CONFIGURE_H_IN)

##____________________________________________________________________
## Build 'libqrencode' library

add_library (qrencode SHARED
  qrencode.c
  qrencode_inner.h
  qrinput.c
  bitstream.c
  qrspec.c
  rsecc.c
  split.c
  mask.c
  mqrspec.c
  mmask.c
  qrenc.c
  qrcodelib.def
  )

set_target_properties (qrencode
  PROPERTIES
  COMPILE_FLAGS "-Wall -g"
  #LINK_FLAGS "-DPIC -version-number ${PROJECT_VERSION}"
)

 target_link_libraries (qrencode
      wingetopt.lib
 )


## Installation instructions

install (
  TARGETS qrencode
  RUNTIME DESTINATION bin
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib
  )

#EXTRA_DIST = libqrencode.pc.in autogen.sh configure.ac acinclude.m4 \
#			 Makefile.am tests/Makefile.am qrencode.spec.in qrencode.spec \
#			 qrencode.1.in Doxyfile tests/test_all.sh

##____________________________________________________________________
## Build utility tools

if (WITH_TOOLS)

  if (PNG_LIBRARIES)
    
    ## Compiler instructions
    add_executable (qrencode_app qrenc.c)
    
    ## Linker instructions
    target_link_libraries (qrencode_app
      qrencode
      ${PNG_LIBRARIES}
      )
    
    ## Extra target properties
    set_target_properties (qrencode_app
      PROPERTIES
      OUTPUT_NAME qrencode_app)
    
    ## Installation instructions
    install (
      TARGETS qrencode_app
      RUNTIME DESTINATION bin
      RENAME qrencode
      )
    
  else (PNG_LIBRARIES)

    message (STATUS "Unable to build tool qrencode - missing PNG library!")
    set (WITH_TOOLS NO)
    
  endif (PNG_LIBRARIES)
  
endif (WITH_TOOLS)

##____________________________________________________________________
## Build test programs

if (WITH_TESTS)
  
  include (CTest)
  enable_testing()
  
  add_subdirectory (tests)
  
endif (WITH_TESTS)

## ==============================================================================
##
##  Documentation
##
## ==============================================================================

if (DOXYGEN_EXECUTABLE)
  
  ## Configuration file for Doxygen
  configure_file (
    ${PROJECT_SOURCE_DIR}/Doxyfile.in
    ${PROJECT_BINARY_DIR}/Doxyfile
    )

  ## Target to build documentation
  add_custom_target (doc
    COMMAND ${DOXYGEN_EXECUTABLE} Doxyfile
    DEPENDS ${PROJECT_BINARY_DIR}/Doxyfile
    WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
    )
  
endif (DOXYGEN_EXECUTABLE)

## ==============================================================================
##
##  Configuration summary
##
## ==============================================================================

message (STATUS "------------------------------------------------------------"   )
message (STATUS "[QRencode] Configuration summary."                              )
message (STATUS "------------------------------------------------------------ "  )
message (STATUS " System configuration:"                                         )
message (STATUS " .. Processor type .............. = ${CMAKE_SYSTEM_PROCESSOR}"  )
message (STATUS " .. CMake executable ............ = ${CMAKE_COMMAND}"           )
message (STATUS " .. CMake version ............... = ${CMAKE_VERSION}"           )
message (STATUS " .. System name ................. = ${CMAKE_SYSTEM}"            )
message (STATUS " .. C++ compiler ................ = ${CMAKE_CXX_COMPILER}"      )
message (STATUS " .. C compiler .................. = ${CMAKE_C_COMPILER}"        )
message (STATUS " .. size(void*) ................. = ${CMAKE_SIZEOF_VOID_P}"     )
message (STATUS " Dependencies:"                                                 )
message (STATUS " .. Doxygen ..................... = ${DOXYGEN_EXECUTABLE}"      )
message (STATUS " .. Thread library of the system  = ${CMAKE_THREAD_LIBS_INIT}"  )
message (STATUS " .. Iconv includes .............. = ${ICONV_INCLUDES}"          )
message (STATUS " .. Iconv library ............... = ${ICONV_LIBRARIES}"         )
message (STATUS " .. PNG includes ................ = ${PNG_INCLUDE_DIR}"         )
message (STATUS " .. PNG library ................. = ${PNG_LIBRARIES}"           )
message (STATUS " .. Memory checker .............. = ${MEMORYCHECK_COMMAND}"     )
message (STATUS " Project configuration:"                                        )
message (STATUS " .. Build test programs? ........ = ${WITH_TESTS}"              )
message (STATUS " .. Build utility tools? ........ = ${WITH_TOOLS}"              )
message (STATUS " .. Installation prefix ......... = ${CMAKE_INSTALL_PREFIX}"    )
message (STATUS "------------------------------------------------------------ "  )

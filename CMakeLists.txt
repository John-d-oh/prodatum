cmake_minimum_required( VERSION 3.21 )

# Must be set before project() so it applies to every sub-project. 11.0 is
# the first macOS release with Apple Silicon support; the resulting binary
# therefore runs on every arm64 Mac (and on Intel Macs when built for x86_64
# or as a universal binary via -DCMAKE_OSX_ARCHITECTURES="arm64;x86_64").
if( APPLE AND NOT CMAKE_OSX_DEPLOYMENT_TARGET )
  set( CMAKE_OSX_DEPLOYMENT_TARGET "11.0" CACHE STRING "Minimum macOS version" FORCE )
endif()

file( STRINGS "PRODATUM_VERSION" PRODATUM_VERSION_IN )

project( prodatum VERSION ${PRODATUM_VERSION_IN} LANGUAGES C CXX )

set( PRODATUM_NAME ${PROJECT_NAME} )
set( PRODATUM_VERSION ${PROJECT_VERSION} )
set( PRODATUM_VERSION_MAJOR ${PROJECT_VERSION_MAJOR} )
set( PRODATUM_VERSION_MINOR ${PROJECT_VERSION_MINOR} )
set( PRODATUM_VERSION_PATCH ${PROJECT_VERSION_PATCH} )

message( STATUS "prodatum name   : [${PRODATUM_NAME}]" )
message( STATUS "prodatum version: [${PRODATUM_VERSION}]" )
if( APPLE )
  message( STATUS "macOS deployment target: ${CMAKE_OSX_DEPLOYMENT_TARGET}" )
  if( CMAKE_OSX_ARCHITECTURES )
    message( STATUS "macOS architectures: ${CMAKE_OSX_ARCHITECTURES}" )
  else()
    message( STATUS "macOS architectures: host (${CMAKE_HOST_SYSTEM_PROCESSOR})" )
  endif()
endif()

if( NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES )
  set( CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE )
endif()

set( CMAKE_CXX_STANDARD 14 )
set( CMAKE_CXX_STANDARD_REQUIRED ON )
set( CMAKE_CXX_EXTENSIONS OFF )

macro( SET_OPTION option value )
  set( ${option} ${value} CACHE INTERNAL "" FORCE )
endmacro()

option( BUILD_LIBS "Build the bundled FLTK and portmidi submodules" ON )
option( PRODATUM_MACOS_BUNDLE "On macOS, build prodatum as a .app bundle" ON )

# --- bundled libraries -------------------------------------------------------
# Both libraries are git submodules tracking upstream releases:
#   lib/fltk     -> https://github.com/fltk/fltk        (release-1.4.x)
#   lib/portmidi -> https://github.com/PortMidi/portmidi (v2.0.x)
# Everything is built static so the resulting executable is self-contained.

SET_OPTION( BUILD_SHARED_LIBS OFF )

# portmidi
SET_OPTION( BUILD_PORTMIDI_TESTS OFF )
SET_OPTION( BUILD_JAVA_NATIVE_INTERFACE OFF )
SET_OPTION( BUILD_DOC OFF )
SET_OPTION( PM_USE_STATIC_RUNTIME OFF )

# fltk
SET_OPTION( FLTK_BUILD_SHARED_LIBS OFF )
SET_OPTION( FLTK_BUILD_TEST OFF )
SET_OPTION( FLTK_BUILD_EXAMPLES OFF )
SET_OPTION( FLTK_BUILD_FLTK_OPTIONS OFF )
SET_OPTION( FLTK_BUILD_FORMS OFF )
SET_OPTION( FLTK_BUILD_GL OFF )
SET_OPTION( FLTK_BUILD_FLUID ON )
SET_OPTION( FLTK_BUILD_HTML_DOCS OFF )
SET_OPTION( FLTK_BUILD_PDF_DOCS OFF )
# Use the copies of zlib/libpng/libjpeg shipped with FLTK so the binary does
# not depend on Homebrew / distro libraries at run time.
SET_OPTION( FLTK_USE_SYSTEM_ZLIB OFF )
SET_OPTION( FLTK_USE_SYSTEM_LIBPNG OFF )
SET_OPTION( FLTK_USE_SYSTEM_LIBJPEG OFF )
if( UNIX AND NOT APPLE )
  # X11 only: keeps the Linux dependency list short (no wayland/libdecor).
  SET_OPTION( FLTK_BACKEND_WAYLAND OFF )
endif()

set( SYSEX_MAX_SIZE 1024 )
set( RINGBUFFER_WRITE 2048 )
set( RINGBUFFER_READ 2048 )
set( PATH_MAX 1024 )
set( LOG_BUFFER_SIZE 1048576 )

configure_file( ${CMAKE_SOURCE_DIR}/cmake/config.h.in ${CMAKE_BINARY_DIR}/include/config.h )

if( BUILD_LIBS )
  if( NOT EXISTS "${CMAKE_SOURCE_DIR}/lib/fltk/CMakeLists.txt" OR
      NOT EXISTS "${CMAKE_SOURCE_DIR}/lib/portmidi/CMakeLists.txt" )
    message( FATAL_ERROR
      "lib/fltk or lib/portmidi is empty. Fetch the submodules first:\n"
      "  git submodule update --init --recursive" )
  endif()
  add_subdirectory( ${CMAKE_SOURCE_DIR}/lib/portmidi EXCLUDE_FROM_ALL )
  add_subdirectory( ${CMAKE_SOURCE_DIR}/lib/fltk EXCLUDE_FROM_ALL )
endif( BUILD_LIBS )

# --- fluid: generate ui.cpp / ui.h from prodatum.fl --------------------------
# FLTK >= 1.4 exports FLTK_FLUID_EXECUTABLE (a target such as fltk::fluid or
# fltk::fluid-cmd depending on the platform).
if( NOT FLTK_FLUID_EXECUTABLE )
  set( FLTK_FLUID_EXECUTABLE fltk::fluid )
endif()

set( UI_AUTOGEN_CPP "${CMAKE_BINARY_DIR}/ui.cpp" )
set( UI_AUTOGEN_H   "${CMAKE_BINARY_DIR}/ui.h" )

add_custom_command(
  OUTPUT ${UI_AUTOGEN_CPP} ${UI_AUTOGEN_H}
  COMMAND ${FLTK_FLUID_EXECUTABLE} -c "${CMAKE_SOURCE_DIR}/prodatum.fl"
  WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
  MAIN_DEPENDENCY "${CMAKE_SOURCE_DIR}/prodatum.fl"
  DEPENDS ${FLTK_FLUID_EXECUTABLE}
  COMMENT "Generating ui.cpp from prodatum.fl with fluid"
  VERBATIM
)

# --- prodatum ----------------------------------------------------------------
set( SOURCES
     src/cfg.cpp
     src/data.cpp
     src/debug.cpp
     src/Fl_Scope.cpp
     src/midi.cpp
     src/prodatum.cpp
     src/pxk.cpp
     src/ringbuffer.cpp
     src/widgets.cpp
)

set( RES_FILES "" )
if( WIN32 )
  set( RES_FILES "resources/windows.rc" )
endif()

if( APPLE AND PRODATUM_MACOS_BUNDLE )
  # Build an .icns from the PNG using tools that ship with macOS.
  set( PRODATUM_ICNS "${CMAKE_BINARY_DIR}/prodatum.icns" )
  set( PRODATUM_ICONSET "${CMAKE_BINARY_DIR}/prodatum.iconset" )
  add_custom_command(
    OUTPUT ${PRODATUM_ICNS}
    COMMAND ${CMAKE_COMMAND} -E rm -rf ${PRODATUM_ICONSET}
    COMMAND ${CMAKE_COMMAND} -E make_directory ${PRODATUM_ICONSET}
    COMMAND sips -z 16 16     "${CMAKE_SOURCE_DIR}/resources/prodatum.png" --out ${PRODATUM_ICONSET}/icon_16x16.png
    COMMAND sips -z 32 32     "${CMAKE_SOURCE_DIR}/resources/prodatum.png" --out ${PRODATUM_ICONSET}/icon_16x16@2x.png
    COMMAND sips -z 32 32     "${CMAKE_SOURCE_DIR}/resources/prodatum.png" --out ${PRODATUM_ICONSET}/icon_32x32.png
    COMMAND sips -z 64 64     "${CMAKE_SOURCE_DIR}/resources/prodatum.png" --out ${PRODATUM_ICONSET}/icon_32x32@2x.png
    COMMAND sips -z 128 128   "${CMAKE_SOURCE_DIR}/resources/prodatum.png" --out ${PRODATUM_ICONSET}/icon_128x128.png
    COMMAND sips -z 256 256   "${CMAKE_SOURCE_DIR}/resources/prodatum.png" --out ${PRODATUM_ICONSET}/icon_128x128@2x.png
    COMMAND sips -z 256 256   "${CMAKE_SOURCE_DIR}/resources/prodatum.png" --out ${PRODATUM_ICONSET}/icon_256x256.png
    COMMAND sips -z 512 512   "${CMAKE_SOURCE_DIR}/resources/prodatum.png" --out ${PRODATUM_ICONSET}/icon_256x256@2x.png
    COMMAND sips -z 512 512   "${CMAKE_SOURCE_DIR}/resources/prodatum.png" --out ${PRODATUM_ICONSET}/icon_512x512.png
    COMMAND sips -z 1024 1024 "${CMAKE_SOURCE_DIR}/resources/prodatum.png" --out ${PRODATUM_ICONSET}/icon_512x512@2x.png
    COMMAND iconutil -c icns ${PRODATUM_ICONSET} -o ${PRODATUM_ICNS}
    DEPENDS "${CMAKE_SOURCE_DIR}/resources/prodatum.png"
    COMMENT "Generating prodatum.icns"
    VERBATIM
  )
  set_source_files_properties( ${PRODATUM_ICNS} PROPERTIES MACOSX_PACKAGE_LOCATION Resources )
  add_executable( prodatum MACOSX_BUNDLE ${SOURCES} ${UI_AUTOGEN_CPP} ${PRODATUM_ICNS} )
  set_target_properties( prodatum PROPERTIES
    MACOSX_BUNDLE_INFO_PLIST            "${CMAKE_SOURCE_DIR}/resources/macos/Info.plist.in"
    MACOSX_BUNDLE_BUNDLE_NAME           "prodatum"
    MACOSX_BUNDLE_GUI_IDENTIFIER        "net.sourceforge.prodatum"
    MACOSX_BUNDLE_ICON_FILE             "prodatum.icns"
    MACOSX_BUNDLE_BUNDLE_VERSION        "${PRODATUM_VERSION}"
    MACOSX_BUNDLE_SHORT_VERSION_STRING  "${PRODATUM_VERSION}"
    MACOSX_BUNDLE_COPYRIGHT             "prodatum contributors, GPL"
  )
else()
  add_executable( prodatum ${SOURCES} ${UI_AUTOGEN_CPP} ${RES_FILES} )
endif()

target_include_directories( prodatum PRIVATE
  ${CMAKE_SOURCE_DIR}
  ${CMAKE_SOURCE_DIR}/include
  ${CMAKE_SOURCE_DIR}/lib/portmidi/pm_common
  ${CMAKE_SOURCE_DIR}/lib/portmidi/porttime
  ${CMAKE_BINARY_DIR}
  ${CMAKE_BINARY_DIR}/include
)

target_compile_definitions( prodatum PRIVATE SYNCLOG )

if( WIN32 )
  target_compile_definitions( prodatum PRIVATE WIN32 )
  if( MINGW )
    set( CMAKE_RC_COMPILER_INIT windres )
    enable_language( RC )
    set( CMAKE_RC_COMPILE_OBJECT "<CMAKE_RC_COMPILER> -O coff <DEFINES> -i <SOURCE> -o <OBJECT>" )
  endif()
  if( "${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU" )
    target_compile_options( prodatum PRIVATE -Wall -W -Wno-char-subscripts -Wno-write-strings -Wno-format )
    target_link_options( prodatum PRIVATE -mwindows -static-libgcc -static-libstdc++ )
    set( ADDITIONAL_LIBRARIES winmm ole32 uuid comctl32 wsock32 )
  elseif( MSVC )
    target_compile_definitions( prodatum PRIVATE _CRT_SECURE_NO_WARNINGS )
    set( ADDITIONAL_LIBRARIES winmm ole32 uuid comctl32 wsock32 gdi32 )
    target_link_options( prodatum PRIVATE "/SUBSYSTEM:WINDOWS" "/ENTRY:mainCRTStartup" )
  endif()
elseif( APPLE )
  target_compile_definitions( prodatum PRIVATE USE_MLOCK OSX )
  target_compile_options( prodatum PRIVATE -Wall -W -Wno-char-subscripts -Wno-write-strings )
  # fltk::fltk and portmidi carry their own framework dependencies.
  set( ADDITIONAL_LIBRARIES "" )
elseif( CMAKE_SYSTEM_NAME MATCHES "Linux" )
  target_compile_definitions( prodatum PRIVATE USE_MLOCK __linux )
  target_compile_options( prodatum PRIVATE -Wall -W -Wno-char-subscripts -Wno-write-strings )
  target_link_options( prodatum PRIVATE -static-libgcc -static-libstdc++ )
  set( ADDITIONAL_LIBRARIES pthread )
endif()

target_link_libraries( prodatum PRIVATE fltk::fltk portmidi ${ADDITIONAL_LIBRARIES} )

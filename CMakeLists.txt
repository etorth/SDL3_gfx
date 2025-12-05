#Sets the minimum required version of cmake for a project
cmake_minimum_required(VERSION 3.16.0)

# Set the project name
project(SDL3_gfx LANGUAGES C VERSION "1.0.1")
set(SDL_REQUIRED_VERSION 3.2.0)
string(REPLACE "." ";" SO_VERSION_COMPONENTS "${PROJECT_VERSION}")
list(GET SO_VERSION_COMPONENTS 0 SO_VERSION_MAJOR)

################################################################################
# Options
################################################################################
include(GNUInstallDirs)

option(BUILD_SHARED_LIBS "Build shared library" ON)
option(BUILD_TESTS "Build tests" Off)

################################################################################
# Source groups
################################################################################
# Assign list of header files to the variable 'Headers'
set(Headers
    "SDL3_framerate.h"
    "SDL3_gfxPrimitives.h"
    "SDL3_gfxPrimitives_font.h"
    "SDL3_imageFilter.h"
    "SDL3_rotozoom.h"
)

#Define a grouping for source files (for IDEs)
source_group("Headers" FILES ${Headers})

# Assign list of source files to the variable 'Sources'
set(Sources
    "SDL3_framerate.c"
    "SDL3_gfxPrimitives.c"
    "SDL3_imageFilter.c"
    "SDL3_rotozoom.c"
)

#Define a grouping for source files (for IDEs)
source_group("Sources" FILES ${Sources})

set(ALL_FILES
    ${Headers}
    ${Sources}
)

################################################################################
# Target
################################################################################
# Build an static and a dynamic library.
if(BUILD_SHARED_LIBS AND NOT (CMAKE_SYSTEM_NAME STREQUAL "Emscripten"))
  add_library(${PROJECT_NAME}-shared SHARED ${ALL_FILES})
  add_library(SDL3_gfx::SDL3_gfx-shared ALIAS ${PROJECT_NAME}-shared)
endif()
add_library(${PROJECT_NAME}-static STATIC ${ALL_FILES})
add_library(SDL3_gfx::SDL3_gfx-static ALIAS ${PROJECT_NAME}-static)

# Create SDL3_gfx::SDL3_gfx alias based on BUILD_SHARED_LIBS
if(BUILD_SHARED_LIBS AND NOT (CMAKE_SYSTEM_NAME STREQUAL "Emscripten"))
  add_library(SDL3_gfx::SDL3_gfx ALIAS ${PROJECT_NAME}-shared)
else()
  add_library(SDL3_gfx::SDL3_gfx ALIAS ${PROJECT_NAME}-static)
endif()

# Set 3 properties on the static target
set_target_properties(
    ${PROJECT_NAME}-static PROPERTIES
  OUTPUT_NAME ${PROJECT_NAME}
  ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_NAME}_Static
  PUBLIC_HEADER "${Headers}"
)
if(BUILD_SHARED_LIBS AND NOT(CMAKE_SYSTEM_NAME STREQUAL "Emscripten"))
  set_target_properties(
      ${PROJECT_NAME}-shared PROPERTIES
    VERSION "${CMAKE_PROJECT_VERSION}" SOVERSION "${SO_VERSION_MAJOR}"
    OUTPUT_NAME ${PROJECT_NAME}
    LIBRARY_OUTPUT_DIRECTORY ${PROJECT_NAME}_Shared
  )
endif()

################################################################################
# Include directories
################################################################################
# Specify include directories to use when compiling SDL3_gfx
# Include SDL3 include folder
if(BUILD_SHARED_LIBS AND NOT (CMAKE_SYSTEM_NAME STREQUAL "Emscripten"))
  target_include_directories(${PROJECT_NAME}-shared PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/SDL3_gfx>
  )
endif()

# Add the include directories of standard SDL3 for static library.
target_include_directories(${PROJECT_NAME}-static PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/SDL3_gfx>
)

################################################################################
# Dependencies
################################################################################

# In Emscripten there are only static libraries.
if(CMAKE_SYSTEM_NAME STREQUAL "Emscripten")
  find_library(SDL3_LIB libSDL3.a)
else()
  find_package(SDL3 REQUIRED)
endif()

# Specify libraries directories that we will link with SDL3_gfx.
if(BUILD_SHARED_LIBS AND NOT(CMAKE_SYSTEM_NAME STREQUAL "Emscripten"))
  target_link_libraries(${PROJECT_NAME}-shared PUBLIC SDL3::SDL3)
endif()
target_link_libraries(${PROJECT_NAME}-static PUBLIC SDL3::SDL3)

# Install targets with EXPORT for CMake config generation
if(BUILD_SHARED_LIBS AND NOT(CMAKE_SYSTEM_NAME STREQUAL "Emscripten"))
  install(TARGETS ${PROJECT_NAME}-shared
    EXPORT SDL3_gfxTargets
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  )
endif()
# Install headers following the convention of SDL3.
install(TARGETS ${PROJECT_NAME}-static
  EXPORT SDL3_gfxTargets
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/SDL3_gfx
)

# Export targets for build tree
if(BUILD_SHARED_LIBS AND NOT(CMAKE_SYSTEM_NAME STREQUAL "Emscripten"))
  export(TARGETS ${PROJECT_NAME}-shared ${PROJECT_NAME}-static
    NAMESPACE SDL3_gfx::
    FILE "${CMAKE_CURRENT_BINARY_DIR}/SDL3_gfxTargets.cmake"
  )
else()
  export(TARGETS ${PROJECT_NAME}-static
    NAMESPACE SDL3_gfx::
    FILE "${CMAKE_CURRENT_BINARY_DIR}/SDL3_gfxTargets.cmake"
  )
endif()

################################################################################
# CMake config files
################################################################################
include(CMakePackageConfigHelpers)

# Determine where to install CMake config files
if(WIN32 AND NOT MINGW)
  set(SDL3_GFX_INSTALL_CMAKEDIR_ROOT_DEFAULT "cmake")
else()
  set(SDL3_GFX_INSTALL_CMAKEDIR_ROOT_DEFAULT "${CMAKE_INSTALL_LIBDIR}/cmake")
endif()
set(SDL3_GFX_INSTALL_CMAKEDIR_ROOT "${SDL3_GFX_INSTALL_CMAKEDIR_ROOT_DEFAULT}" CACHE STRING "Root folder where to install SDL3_gfxConfig.cmake related files")

if(WIN32 AND NOT MINGW)
  set(SDL3_GFX_INSTALL_CMAKEDIR "${SDL3_GFX_INSTALL_CMAKEDIR_ROOT}")
else()
  set(SDL3_GFX_INSTALL_CMAKEDIR "${SDL3_GFX_INSTALL_CMAKEDIR_ROOT}/SDL3_gfx")
endif()

# Generate and install CMake config files
configure_package_config_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/SDL3_gfxConfig.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/SDL3_gfxConfig.cmake"
  INSTALL_DESTINATION "${SDL3_GFX_INSTALL_CMAKEDIR}"
  NO_SET_AND_CHECK_MACRO
)

write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/SDL3_gfxConfigVersion.cmake"
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY AnyNewerVersion
)

install(
  FILES
    "${CMAKE_CURRENT_BINARY_DIR}/SDL3_gfxConfig.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/SDL3_gfxConfigVersion.cmake"
  DESTINATION "${SDL3_GFX_INSTALL_CMAKEDIR}"
)

install(
  EXPORT SDL3_gfxTargets
  FILE SDL3_gfxTargets.cmake
  NAMESPACE SDL3_gfx::
  DESTINATION "${SDL3_GFX_INSTALL_CMAKEDIR}"
)

################################################################################
# pkg-config
################################################################################
configure_file(sdl3-gfx.pc.in sdl3-gfx.pc @ONLY)
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/sdl3-gfx.pc"
  DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig")


################################################################################
# Sub-projects
################################################################################
# build the test also
if(BUILD_TESTS AND NOT CMAKE_SYSTEM_NAME STREQUAL "Emscripten")
  add_subdirectory(test)
endif()

cmake_minimum_required (VERSION 3.20)

project (XEVE VERSION 0.5.0 LANGUAGES C)

option(XEVE_BUILD_APP "Build xeve app" OFF)
option(XEVE_INSTALL "Install xeve" OFF)

set(XEVE_PROFILE "MAIN" CACHE STRING "xeve profile (BASE/MAIN)")
set_property(CACHE XEVE_PROFILE PROPERTY STRINGS BASE MAIN)

if(NOT DEFINED XEVE_PROFILE)
    set(XEVE_PROFILE "MAIN")
elseif (NOT (XEVE_PROFILE STREQUAL "MAIN") AND NOT (XEVE_PROFILE STREQUAL "BASE"))
    message(FATAL_ERROR "XEVE_PROFILE should be BASE or MAIN [Input error: ${XEVE_PROFILE}]")
endif()

message(STATUS "XEVE_PROFILE = ${XEVE_PROFILE}")

# To build for arm provide in command line: -DARM=TRUE
if(NOT ARM)
  set(ARM "FALSE")
else()
  add_definitions(-DARM=1)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -flax-vector-conversions")
  if(NOT "${CMAKE_SYSTEM_NAME}" STREQUAL "Darwin")
    set(CMAKE_EXE_LINKER_FLAGS "-static")
  endif()
endif()

########################################
# Compilation flags
########################################

# Set compiler flags and options.
if( MSVC )
    add_compile_options(
        $<$<CONFIG:>:/MD> #---------|
        $<$<CONFIG:Debug>:/MDd> #---|-- Dynamically link the runtime libraries
        $<$<CONFIG:Release>:/MD> #--|
    )
elseif("${CMAKE_C_COMPILER_ID}" STREQUAL "Clang")
    if("${CMAKE_BUILD_TYPE}" STREQUAL "Debug")
        set(OPT_LV "O0")
        set(OPT_DBG "-g")
    else()
        set(OPT_LV "O3")
        set(OPT_DBG "-DNDEBUG") # disable assert
    endif()

    set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OPT_DBG} -${OPT_LV} -fomit-frame-pointer -pthread -std=c99")
    set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wno-pointer-sign -Wno-unused-function -Wno-unused-variable")
    set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -lm")
elseif( UNIX OR MINGW )
    if("${CMAKE_BUILD_TYPE}" STREQUAL "Debug")
        set(OPT_LV "O0")
        set(OPT_DBG "-g")
    else()
        set(OPT_LV "O3")
        set(OPT_DBG "-DNDEBUG") # disable assert
    endif()

    set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OPT_DBG} -${OPT_LV} -fomit-frame-pointer -pthread -std=c99")
    #TODO: fix suppressed problems
    set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Werror -Wno-pointer-sign -Wno-unused-function -Wno-unused-but-set-variable -Wno-unused-variable -Wno-attributes -Wno-strict-overflow -Wno-unknown-pragmas -Wno-stringop-overflow -Wno-pointer-to-int-cast -Wno-maybe-uninitialized")
    set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -lm")
endif()

########################################
# Configuration
########################################

set(CMAKE_C_STANDARD 99)
set_property(GLOBAL PROPERTY USE_FOLDERS ON)
add_definitions(-Dxeve_dynamic_EXPORTS)
cmake_policy(SET CMP0048 NEW)

# Sub-directories where more CMakeLists.txt exist
set( LIB_NAME xeve)
set( LIB_NAME_BASE xeveb)

if(XEVE_PROFILE STREQUAL "BASE")
    add_subdirectory(src_base)
endif()
if(XEVE_PROFILE STREQUAL "MAIN")
    add_subdirectory(src_main)
endif()
if (XEVE_BUILD_APP)
    add_subdirectory(app)
endif ()

########################################
# CPack project packaging
########################################

if( MSVC )
  set(CPACK_GENERATOR "NSIS")
elseif( UNIX )
  set(CPACK_GENERATOR "DEB")
endif()

# Package name
if(XEVE_PROFILE STREQUAL "BASE")
    set(PACKAGE_PROFILE_NAME "base")
endif()

if(XEVE_PROFILE STREQUAL "MAIN")
    set(PACKAGE_PROFILE_NAME "main")
endif()

if (XEVE_INSTALL)

    string(TOLOWER "${PROJECT_NAME}" PACKAGE_NAME)
    string(CONCAT PACKAGE_NAME ${PACKAGE_NAME} "-${PACKAGE_PROFILE_NAME}")

    set(CPACK_PACKAGE_NAME ${PACKAGE_NAME})

    # Package License & Readme file
    set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/COPYING")
    set(CPACK_RESOURCE_FILE_README "${CMAKE_CURRENT_SOURCE_DIR}/README.md")

    # Package checksum
    set(CPACK_PACKAGE_CHECKSUM MD5)

    # Short description of the project
    set(CPACK_PACKAGE_DESCRIPTION_SUMMARY
    "eXtra-fast Essential Video Encoder, MPEG-5 EVC (Essential Video Coding), ${PACKAGE_PROFILE_NAME} profile")

    # Project homepage
    set(CPACK_PACKAGE_VENDOR "MPEG-5")
    set(CPACK_PACKAGE_HOMEPAGE_URL "https://github.com/mpeg5/xeve/releases")
    set(CMAKE_PROJECT_HOMEPAGE_URL "https://github.com/mpeg5/xeve")

    # Installation directory on the target system
    string(TOLOWER "${PROJECT_NAME}" PROJECT_NAME_LOWER)
    set(PACKAGE_INSTALL_DIRECTORY "${PROJECT_NAME_LOWER} ${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}")
    set(CPACK_PACKAGE_INSTALL_DIRECTORY "${PACKAGE_INSTALL_DIRECTORY}")

    if( UNIX )
      set(CPACK_DEBIAN_PACKAGE_MAINTAINER "MPEG-Five") #required​
      set(CPACK_PACKAGE_CONTACT "mpegfive@gmail.com") #required

      set(CPACK_DEB_COMPONENT_INSTALL ON)

      # If enabled (ON) multiple packages are generated.
      # By default a single package containing files of all components is generated.
      set(CPACK_DEB_COMPONENT_INSTALL YES)

      # This may be set to DEB-DEFAULT to allow CPackDeb to generate package file name by itself in deb format:
      # <PackageName>_<VersionNumber>-<DebianRevisionNumber>_<DebianArchitecture>.deb
      set(CPACK_DEBIAN_FILE_NAME DEB-DEFAULT)

      set(CPACK_DEBIAN_RUNTIMEGROUP_PACKAGE_NAME "${CPACK_PACKAGE_NAME}")
      set(CPACK_DEBIAN_DEVELOPMENTGROUP_PACKAGE_NAME "${CPACK_PACKAGE_NAME}-dev")
      set(CPACK_DEBIAN_DEVELOPMENTGROUP_PACKAGE_DEPENDS ${CPACK_PACKAGE_NAME})
    endif()

    set(CPACK_COMPONENTS_ALL Runtime Libraries Development)
    set(CPACK_COMPONENT_DEVELOPMENT_DEPENDS Libraries)
    set(CPACK_COMPONENT_RUNTIME_DEPENDS Libraries)

    include(CPack)

    if( MSVC )
      cpack_add_install_type(Full DISPLAY_NAME "All")
      cpack_add_install_type(Developer DISPLAY_NAME "Developer files")
    endif()

    cpack_add_component_group(DevelopmentGroup
      DISPLAY_NAME "Development"
      EXPANDED
      DESCRIPTION
      "All of the tools you'll ever need to develop libxeve dependent software")

    cpack_add_component_group(RuntimeGroup
      DISPLAY_NAME "Runtime"
      EXPANDED
      DESCRIPTION
     "Shared library and xeve_app reference application")

    cpack_add_component(Runtime DISPLAY_NAME "Application" DESCRIPTION  "Reference application that makes use of libxeve" GROUP RuntimeGroup INSTALL_TYPES Full Runtime)
    cpack_add_component(Libraries DISPLAY_NAME "Shared Libraries" DESCRIPTION "Shared libraries used to run xeve_app" GROUP RuntimeGroup INSTALL_TYPES Full Runtime Developer)
    cpack_add_component(Development DISPLAY_NAME "C++ Headers and static libraries used to build programs with libxeve" DESCRIPTION "C/C++ header files for use with libxeve" GROUP DevelopmentGroup INSTALL_TYPES Full Developer)

endif ()

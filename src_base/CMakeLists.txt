
file (GLOB XEVE_INC "../inc/*.h")
file (GLOB LIB_API_SRC "./xeve.c")
file (GLOB LIB_BASE_SRC "./xeve_*.c")
file (GLOB LIB_BASE_INC "./xeve_*.h" )
file (GLOB LIB_SSE_SRC "./sse/xeve_*.c")
file (GLOB LIB_SSE_INC "./sse/xeve_*.h" )
file (GLOB LIB_AVX_SRC "./avx/xeve_*.c")
file (GLOB LIB_AVX_INC "./avx/xeve_*.h" )
file (GLOB LIB_NEON_SRC "./neon/xeve_*.c")
file (GLOB LIB_NEON_INC "./neon/xeve_*.h" )

include(GenerateExportHeader)
include_directories("${CMAKE_CURRENT_BINARY_DIR}")

add_library(${LIB_NAME_BASE} STATIC ${LIB_API_SRC} ${XEVE_INC} ${LIB_BASE_SRC} ${LIB_BASE_INC})

add_library(XEVE::${LIB_NAME_BASE} ALIAS ${LIB_NAME_BASE})

if("${ARM}" STREQUAL "TRUE")
  target_sources(${LIB_NAME_BASE} PRIVATE ${LIB_NEON_INC} ${LIB_NEON_SRC})
else()
  target_sources(${LIB_NAME_BASE} PRIVATE ${LIB_SSE_SRC} ${LIB_SSE_INC} ${LIB_AVX_SRC} ${LIB_AVX_INC})
endif()

if(NOT MSVC AND CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64|AMD64")
    target_compile_options(${LIB_NAME_BASE} PRIVATE
            -mssse3
            -msse4.1
            -msse4.2
            -mavx
            -mavx2
            -mavx512f
            -fno-tree-vectorize
    )
endif ()

source_group("base\\header" FILES ${LIB_BASE_INC} ${XEVE_INC})
source_group("base\\source" FILES ${LIB_BASE_SRC} ${LIB_API_SRC})
source_group("base\\sse\\header" FILES ${LIB_SSE_INC})
source_group("base\\sse\\source" FILES ${LIB_SSE_SRC})
source_group("base\\avx\\header" FILES ${LIB_AVX_INC})
source_group("base\\avx\\source" FILES ${LIB_AVX_SRC})
source_group("base\\neon\\header" FILES ${LIB_NEON_INC})
source_group("base\\neon\\source" FILES ${LIB_NEON_SRC})

target_include_directories(${LIB_NAME_BASE} PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/../inc>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

if("${ARM}" STREQUAL "TRUE")
    include_directories( ${LIB_NAME_BASE} PUBLIC ./neon)
else()
    include_directories( ${LIB_NAME_BASE} PUBLIC ./sse ./avx)
endif()

set( SSE ${BASE_INC_FILES} ${LIB_SSE_SRC})
set( AVX ${LIB_AVX_SRC} )
set( NEON ${LIB_NEON_SRC})

set_target_properties(${LIB_NAME_BASE}_dynamic PROPERTIES OUTPUT_NAME ${LIB_NAME_BASE})

if( MSVC )
    target_compile_definitions( ${LIB_NAME_BASE} PUBLIC ANY _CRT_SECURE_NO_WARNINGS )

    # Since both the import library associated with DLL and the static library have the same names
    # they must be build in different locations to avoid overwriting.
    #
    set_target_properties(${LIB_NAME_BASE} PROPERTIES FOLDER lib
                            ARCHIVE_OUTPUT_DIRECTORY  ${CMAKE_CURRENT_BINARY_DIR}/static/lib)

elseif( UNIX OR MINGW )
  if("${ARM}" STREQUAL "FALSE")
    set_property( SOURCE ${SSE} APPEND PROPERTY COMPILE_FLAGS "-msse4.1" )
    set_property( SOURCE ${AVX} APPEND PROPERTY COMPILE_FLAGS " -mavx2" )
  endif()
  target_compile_definitions( ${LIB_NAME_BASE} PUBLIC ANY LINUX )
  target_link_libraries(${LIB_NAME_BASE} m)
endif()

# Install rules
#
# Set DCMAKE_INSTALL_PREFIX to change default install prefix
# e.g cmake .. -DSET_PROF=BASE -DCMAKE_INSTALL_PREFIX='C:\Users\name\git\xeve_internal\build-windows\install'

# List the headers we want to declare as public for installation.
set(XEVE_PUBLIC_HEADERS "${XEVE_INC}")
set_target_properties(${LIB_NAME_BASE} PROPERTIES PUBLIC_HEADER "${XEVE_PUBLIC_HEADERS}")

set(XEVE_PRIVATE_HEADERS "${LIB_BASE_INC}" "${LIB_SSE_INC}" "${LIB_AVX_INC}" "${LIB_NEON_INC}")

if (XEVE_INSTALL)

    include(GNUInstallDirs)

    # Install library and public headers
    install(TARGETS ${LIB_NAME_BASE}
            EXPORT XEVETargets
            ARCHIVE COMPONENT Development DESTINATION ${CMAKE_INSTALL_LIBDIR}
            LIBRARY COMPONENT Libraries DESTINATION ${CMAKE_INSTALL_LIBDIR}
            RUNTIME COMPONENT Libraries DESTINATION ${CMAKE_INSTALL_BINDIR}
            PUBLIC_HEADER COMPONENT Development DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
            INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    )

    configure_file(
            "${CMAKE_CURRENT_SOURCE_DIR}/pkgconfig/${LIB_NAME_BASE}.pc.in"
            "${CMAKE_CURRENT_BINARY_DIR}/${LIB_NAME_BASE}.pc"
            IMMEDIATE @ONLY)

    install( FILES
            "${CMAKE_CURRENT_BINARY_DIR}/${LIB_NAME_BASE}.pc" COMPONENT Development DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
    )

    # Install the export target
    install(EXPORT XEVETargets
            FILE XEVETargets.cmake
            NAMESPACE XEVE::
            DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/XEVE
            COMPONENT Development
    )

endif ()

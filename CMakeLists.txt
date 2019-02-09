cmake_minimum_required(VERSION 3.1.0)

project(FLAC)

enable_language(C)
enable_language(CXX)

set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

option(BUILD_SHARED_LIBS "Build shared libraries" OFF)
option(SKIP_INSTALL_ALL "Skip install all" OFF)
option(ENABLE_OGG "Enbale build with OGG library" OFF)

set(PACKAGE_NAME "flac")
set(PACKAGE_VERSION "1.3.2")
set(PACKAGE_STRING "${PACKAGE_NAME} ${PACKAGE_VERSION}")
set(FLAC_LIB_VERSION "8.3.0")
set(FLAC_LIB_SOVERSION 8)
set(FLAC++_LIB_VERSION "6.3.0")
set(FLAC++_LIB_SOVERSION 6)

###############################################################################
# configure

include(CheckFunctionExists)
include(CheckIncludeFile)
include(CheckTypeSize)
include(CheckSymbolExists)
include(TestBigEndian)

check_type_size       ("int"            SIZEOF_INT)
check_type_size       ("off_t"          SIZEOF_OFF_T)
check_include_file    ("stdint.h"       HAVE_STDINT_H)
check_include_file    ("byteswap.h"     HAVE_BYTESWAP_H)
check_include_file    ("cpuid.h"        HAVE_CPUID_H)
check_include_file    ("inttypes.h"     HAVE_INTTYPES_H)
check_include_file    ("unistd.h"       HAVE_UNISTD_H)
check_include_file    ("sys/param.h"    HAVE_SYS_PARAM_H)
check_symbol_exists   ("lround"         "math.h;tgmath.h" HAVE_LROUND)
check_symbol_exists   ("bswap_16"       "byteswap.h" HAVE_BSWAP16)
check_symbol_exists   ("bswap_32"       "byteswap.h" HAVE_BSWAP32)
check_symbol_exists   ("bswap_64"       "byteswap.h" HAVE_BSWAP64)
check_symbol_exists   ("iconv"          "iconv.h" HAVE_ICONV)
check_function_exists ("clock_gettime"  HAVE_CLOCK_GETTIME)
check_function_exists ("getopt_long"    HAVE_GETOPT_LONG)

test_big_endian(CPU_IS_BIG_ENDIAN)
if(${CPU_IS_BIG_ENDIAN} EQUAL 0)
    set(CPU_IS_LITTLE_ENDIAN 1)
else()
    set(CPU_IS_LITTLE_ENDIAN 0)
endif()

set(FLAC__NO_ASM 1)

if(ENABLE_OGG)
    set(FLAC__HAS_OGG 1)
else()
    set(FLAC__HAS_OGG 0)
endif()

if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(FLAC__SYS_DARWIN 1)
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    set(FLAC__SYS_LINUX 1)
endif()

if (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "x86_64")
    set(FLAC__CPU_X86_64 1)
elseif (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "amd64")
    set(FLAC__CPU_X86_64 1)
elseif (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "x86")
    set(FLAC__CPU_IA32 1)
elseif (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "i386")
    set(FLAC__CPU_IA32 1)
elseif (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "i686")
    set(FLAC__CPU_IA32 1)
endif()

add_definitions(-DHAVE_CONFIG_H)
configure_file ("${CMAKE_CURRENT_SOURCE_DIR}/cmake/config.h.in"
                "${CMAKE_CURRENT_BINARY_DIR}/config.h" )

include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/src/libFLAC/include
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${CMAKE_CURRENT_BINARY_DIR}
)

set(
    FLAC_SRCS
    src/libFLAC/bitmath.c
    src/libFLAC/bitreader.c
    src/libFLAC/bitwriter.c
    src/libFLAC/cpu.c
    src/libFLAC/crc.c
    src/libFLAC/fixed.c
    src/libFLAC/fixed_intrin_sse2.c
    src/libFLAC/fixed_intrin_ssse3.c
    src/libFLAC/float.c
    src/libFLAC/format.c
    src/libFLAC/lpc.c
    src/libFLAC/lpc_intrin_avx2.c
    src/libFLAC/lpc_intrin_sse2.c
    src/libFLAC/lpc_intrin_sse41.c
    src/libFLAC/lpc_intrin_sse.c
    src/libFLAC/md5.c
    src/libFLAC/memory.c
    src/libFLAC/metadata_iterators.c
    src/libFLAC/metadata_object.c
    src/libFLAC/stream_decoder.c
    src/libFLAC/stream_encoder.c
    src/libFLAC/stream_encoder_framing.c
    src/libFLAC/stream_encoder_intrin_avx2.c
    src/libFLAC/stream_encoder_intrin_sse2.c
    src/libFLAC/stream_encoder_intrin_ssse3.c
    src/libFLAC/window.c
)

set(
    windows_unicode_compat_SRCS
    src/libFLAC/windows_unicode_filenames.c
)

set(
    extra_ogg_SRCS
    src/libFLAC/ogg_decoder_aspect.c
    src/libFLAC/ogg_encoder_aspect.c
    src/libFLAC/ogg_helper.c
    src/libFLAC/ogg_mapping.c
)

if(NOT MSVC)
    set(FLAC_link_libraries m)
else()
    set(extra_SRCS ${windows_unicode_compat_SRCS})
    set(FLAC_link_libraries ws2_32)
endif()

if(ENABLE_OGG)
    find_library(ogg_LIBRARIES ogg)
    list(APPEND extra_SRCS ${extra_ogg_SRCS})
    list(APPEND FLAC_link_libraries ogg)
endif()

add_library(FLAC STATIC ${FLAC_SRCS} ${extra_SRCS})
target_link_libraries(FLAC ${FLAC_link_libraries})
set(FLAC_TGTS FLAC)
if(BUILD_SHARED_LIBS)
    add_library(FLAC-shared SHARED ${FLAC_SRCS} ${extra_SRCS})
    target_link_libraries(FLAC-shared ${FLAC_link_libraries})
    set_target_properties (FLAC-shared PROPERTIES
        VERSION "${FLAC_LIB_VERSION}"
        SOVERSION "${FLAC_LIB_SOVERSION}"
        OUTPUT_NAME FLAC
    )
    set(FLAC_TGTS ${FLAC_TGTS} FLAC-shared)
endif()

set(FLAC_LIBRARIES
    FLAC ${FLAC_link_libraries}
    CACHE INTERNAL "" FORCE
)
set(FLAC_INCLUDE_DIR
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    CACHE INTERNAL "" FORCE
)

set(
    FLAC++_SRCS
    src/libFLAC++/metadata.cpp
    src/libFLAC++/stream_decoder.cpp
    src/libFLAC++/stream_encoder.cpp
)

add_library(FLAC++ STATIC ${FLAC++_SRCS})
add_dependencies(FLAC++ FLAC)
target_link_libraries(FLAC++ FLAC ${FLAC_link_libraries})
set(FLAC++_TGTS FLAC++)
if(BUILD_SHARED_LIBS)
    add_library(FLAC++-shared SHARED ${FLAC++_SRCS})
    add_dependencies(FLAC++-shared FLAC-shared)
    target_link_libraries(FLAC++-shared FLAC-shared ${FLAC_link_libraries})
    set_target_properties (FLAC++-shared PROPERTIES
        VERSION "${FLAC++_LIB_VERSION}"
        SOVERSION "${FLAC++_LIB_SOVERSION}"
        OUTPUT_NAME FLAC++
    )
    set(FLAC++_TGTS ${FLAC++_TGTS} FLAC++-shared)
endif()

set(FLAC++_LIBRARIES
    FLAC++ ${FLAC_LIBRARIES}
    CACHE INTERNAL "" FORCE
)
set(FLAC++_INCLUDE_DIR
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    CACHE INTERNAL "" FORCE
)

if(NOT MSVC)
###############################################################################
# install targets
# Create pkg-conf file.
# We use the same files as ./configure does, so we have to defined the same
# variables used there.

if(NOT SKIP_INSTALL_ALL)

    configure_file(
        "${CMAKE_CURRENT_SOURCE_DIR}/cmake/cmake_uninstall.cmake.in"
        "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
        IMMEDIATE @ONLY)

    add_custom_target(uninstall
        COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)

    if (NOT DEFINED CMAKE_INSTALL_BINDIR)
        set (CMAKE_INSTALL_BINDIR ${CMAKE_INSTALL_PREFIX}/bin)
    endif ()
    if (NOT DEFINED CMAKE_INSTALL_FULL_LIBDIR)
        set (CMAKE_INSTALL_FULL_LIBDIR ${CMAKE_INSTALL_PREFIX}/lib)
    endif ()
    if (NOT DEFINED CMAKE_INSTALL_INCLUDEDIR)
        set (CMAKE_INSTALL_INCLUDEDIR ${CMAKE_INSTALL_PREFIX}/include)
    endif ()
    set (prefix      "${CMAKE_INSTALL_PREFIX}")
    set (exec_prefix "${CMAKE_INSTALL_PREFIX}")
    set (includedir  "${CMAKE_INSTALL_INCLUDEDIR}")
    set (libdir      "${CMAKE_INSTALL_FULL_LIBDIR}")

    ###########################################################################
    # Add FLAC targets to the build-tree export set

    export(TARGETS ${FLAC_TGTS} FILE "${CMAKE_CURRENT_BINARY_DIR}/FLACTargets.cmake")

    configure_file (cmake/FLAC.pc.in FLAC.pc @ONLY)
    configure_file (cmake/FLACConfig.cmake.in FLACConfig.cmake @ONLY)
    configure_file (cmake/FLACConfigVersion.cmake.in FLACConfigVersion.cmake @ONLY)

    install (TARGETS ${FLAC_TGTS}
        EXPORT FLACTargets
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_FULL_LIBDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_FULL_LIBDIR})

    install (FILES
        "${CMAKE_CURRENT_BINARY_DIR}/FLAC.pc"
        DESTINATION "${CMAKE_INSTALL_FULL_LIBDIR}/pkgconfig")

    # Install the FLACConfig.cmake and FLACConfigVersion.cmake
    set(INSTALL_CMAKE_DIR "${CMAKE_INSTALL_FULL_LIBDIR}/cmake/FLAC")
    install (FILES
        "${CMAKE_CURRENT_BINARY_DIR}/FLACConfig.cmake"
        "${CMAKE_CURRENT_BINARY_DIR}/FLACConfigVersion.cmake"
        DESTINATION "${INSTALL_CMAKE_DIR}" COMPONENT dev)

    # Install the export set for use with the install-tree
    install(EXPORT FLACTargets DESTINATION "${INSTALL_CMAKE_DIR}" COMPONENT dev)

    file (GLOB FLAC_HEADER_FILES ${CMAKE_CURRENT_SOURCE_DIR}/include/FLAC/*.h)
    install (FILES ${FLAC_HEADER_FILES}
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/FLAC/)

    ###########################################################################
    # Add FLAC++ targets to the build-tree export set

    export(TARGETS ${FLAC++_TGTS} FILE "${CMAKE_CURRENT_BINARY_DIR}/FLAC++Targets.cmake")

    configure_file (cmake/FLAC++.pc.in FLAC++.pc @ONLY)
    configure_file (cmake/FLAC++Config.cmake.in FLAC++Config.cmake @ONLY)
    configure_file (cmake/FLAC++ConfigVersion.cmake.in FLAC++ConfigVersion.cmake @ONLY)

    install (TARGETS ${FLAC++_TGTS}
        EXPORT FLAC++Targets
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_FULL_LIBDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_FULL_LIBDIR})

    install (FILES
        "${CMAKE_CURRENT_BINARY_DIR}/FLAC++.pc"
        DESTINATION "${CMAKE_INSTALL_FULL_LIBDIR}/pkgconfig")

    # Install the FLAC++Config.cmake and FLAC++ConfigVersion.cmake
    set(INSTALL_CMAKE_DIR "${CMAKE_INSTALL_FULL_LIBDIR}/cmake/FLAC++")
    install (FILES
        "${CMAKE_CURRENT_BINARY_DIR}/FLAC++Config.cmake"
        "${CMAKE_CURRENT_BINARY_DIR}/FLAC++ConfigVersion.cmake"
        DESTINATION "${INSTALL_CMAKE_DIR}" COMPONENT dev)

    # Install the export set for use with the install-tree
    install(EXPORT FLAC++Targets DESTINATION "${INSTALL_CMAKE_DIR}" COMPONENT dev)

    file (GLOB FLAC++_HEADER_FILES ${CMAKE_CURRENT_SOURCE_DIR}/include/FLAC++/*.h)
    install(FILES ${FLAC++_HEADER_FILES}
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/FLAC++/)

endif()
endif()
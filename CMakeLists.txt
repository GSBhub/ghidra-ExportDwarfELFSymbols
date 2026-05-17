cmake_minimum_required(VERSION 3.10)
project(libdwarf_jna C CXX)

include(ExternalProject)

if(WIN32)
    set(OS_SUBDIR "win_x86_64")
    set(SHARED_LIB_FILENAME "libdwarf.dll")
elseif(APPLE)
    set(OS_SUBDIR "mac_x86_64")
    set(SHARED_LIB_FILENAME "libdwarf.dylib")
elseif(UNIX)
    set(OS_SUBDIR "linux_x86_64")
    set(SHARED_LIB_FILENAME "libdwarf.so")
endif()

set(GHIDRA_INSTALL_DIR "/home/scotty/Documents/ghidra_11.4.2_PUBLIC"
    CACHE PATH "Path to the Ghidra installation directory")

option(BUILD_SHARED "Build shared library" ON)
option(BUILD_NON_SHARED "Build static library" OFF)

set(NATIVE_LIB_OUTPUT_DIR "${CMAKE_BINARY_DIR}/os/${OS_SUBDIR}")

# Build libdwarf as an isolated external project so its CMAKE_SOURCE_DIR
# resolves correctly (add_subdirectory breaks this assumption in libdwarf's CMake).
ExternalProject_Add(libdwarf_ext
    SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/libdwarf"
    BINARY_DIR "${CMAKE_BINARY_DIR}/libdwarf-build"
    CMAKE_ARGS
        -DBUILD_SHARED=${BUILD_SHARED}
        -DBUILD_NON_SHARED=${BUILD_NON_SHARED}
        -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
        -DCMAKE_LIBRARY_OUTPUT_DIRECTORY=${NATIVE_LIB_OUTPUT_DIR}
        -DCMAKE_RUNTIME_OUTPUT_DIRECTORY=${NATIVE_LIB_OUTPUT_DIR}
    INSTALL_COMMAND ""
    BUILD_BYPRODUCTS "${NATIVE_LIB_OUTPUT_DIR}/${SHARED_LIB_FILENAME}"
)

# ---- Ghidra packaging -------------------------------------------------------
# Only available when building the shared library (BUILD_SHARED=ON).

if(BUILD_SHARED)
    # Copy the shared library into the source-tree os/<platform>/ directory so
    # that gradle can find and package it in the extension zip.
    add_custom_target(CopyLibdwarf
        COMMAND ${CMAKE_COMMAND} -E make_directory
            "${CMAKE_SOURCE_DIR}/os/${OS_SUBDIR}"
        COMMAND ${CMAKE_COMMAND} -E copy
            "${NATIVE_LIB_OUTPUT_DIR}/${SHARED_LIB_FILENAME}"
            "${CMAKE_SOURCE_DIR}/os/${OS_SUBDIR}/${SHARED_LIB_FILENAME}"
        DEPENDS libdwarf_ext
        COMMENT "Copying native library to os/${OS_SUBDIR}/"
    )

    # Ghidra 11.x requires Java 21. If the environment's JAVA_HOME points to an
    # older JDK (common when the system default is Java 17), the gradlew invocation
    # will fail with UnsupportedClassVersionError. Let the user override via
    # -DGHIDRA_JAVA_HOME=...; otherwise fall back to the environment's JAVA_HOME.
    if(NOT DEFINED GHIDRA_JAVA_HOME OR GHIDRA_JAVA_HOME STREQUAL "")
        set(GHIDRA_JAVA_HOME "$ENV{JAVA_HOME}" CACHE PATH
            "Java 21+ home used when building the Ghidra extension")
    endif()

    set(GRADLE_EXECUTABLE "${GHIDRA_INSTALL_DIR}/support/gradle/gradlew")
    if(EXISTS "${GRADLE_EXECUTABLE}")
        add_custom_target(GhidraExtension ALL
            COMMAND ${CMAKE_COMMAND} -E env "JAVA_HOME=${GHIDRA_JAVA_HOME}"
                "${GRADLE_EXECUTABLE}"
                "-PGHIDRA_INSTALL_DIR=${GHIDRA_INSTALL_DIR}"
                distributeExtension
            WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}"
            DEPENDS CopyLibdwarf
            COMMENT "Building Ghidra extension (gradle distributeExtension)"
        )
    else()
        message(WARNING
            "Ghidra gradlew not found at ${GRADLE_EXECUTABLE} — GhidraExtension target unavailable")
    endif()

    # cmake --install: copy the packaged zip into Ghidra's Extensions directory.
    install(CODE "
        file(GLOB _zips \"${CMAKE_SOURCE_DIR}/dist/*.zip\")
        if(NOT _zips)
            message(FATAL_ERROR
                \"No zip found in dist/. Run cmake --build first.\")
        endif()
        list(SORT _zips ORDER DESCENDING)
        list(GET _zips 0 _zip)
        set(_dest \"${GHIDRA_INSTALL_DIR}/Extensions/Ghidra\")
        file(MAKE_DIRECTORY \"\${_dest}\")
        file(COPY \"\${_zip}\" DESTINATION \"\${_dest}\")
        message(STATUS \"Installed extension to \${_dest}\")
    ")
endif()

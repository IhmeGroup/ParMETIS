cmake_minimum_required(VERSION 3.15)
project(ParMETIS LANGUAGES C)

# Use C++17 in main project compatibility
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# C99 is sufficient for ParMETIS
set(CMAKE_C_STANDARD 99)

# ============================================================================
# Options
# ============================================================================
option(PARMETIS_USE_INT64 "Use 64-bit integers" OFF)
option(PARMETIS_USE_DOUBLE "Use double precision" ON)

# ============================================================================
# MPI - Reuse from parent project
# ============================================================================
if(NOT TARGET MPI::MPI_C)
    find_package(MPI REQUIRED)
endif()

# ============================================================================
# External dependencies - METIS and GKlib (pre-installed)
# ============================================================================
set(METIS_INSTALL_DIR "$ENV{METIS_INSTALL_DIR}" CACHE PATH "Path to METIS installation")
set(GKLIB_INSTALL_DIR "$ENV{GKLIB_INSTALL_DIR}" CACHE PATH "Path to GKlib installation")

# Validate paths
if(NOT METIS_INSTALL_DIR OR METIS_INSTALL_DIR STREQUAL "")
    message(FATAL_ERROR "METIS_INSTALL_DIR not set. Please set via -DMETIS_INSTALL_DIR=... or export METIS_INSTALL_DIR=...")
endif()

if(NOT GKLIB_INSTALL_DIR OR GKLIB_INSTALL_DIR STREQUAL "")
    message(FATAL_ERROR "GKLIB_INSTALL_DIR not set. Please set via -DGKLIB_INSTALL_DIR=... or export GKLIB_INSTALL_DIR=...")
endif()

# Find libraries
find_library(GKLIB_LIBRARY
    NAMES gklib
    PATHS ${GKLIB_INSTALL_DIR}
    PATH_SUFFIXES lib lib64
    NO_DEFAULT_PATH
    REQUIRED
)

find_library(METIS_LIBRARY
    NAMES metis
    PATHS ${METIS_INSTALL_DIR}
    PATH_SUFFIXES lib lib64
    NO_DEFAULT_PATH
    REQUIRED
)

# Find include directories
find_path(GKLIB_INCLUDE_DIR
    NAMES GKlib.h
    PATHS ${GKLIB_INSTALL_DIR}
    PATH_SUFFIXES include
    NO_DEFAULT_PATH
    REQUIRED
)

find_path(METIS_INCLUDE_DIR
    NAMES metis.h
    PATHS ${METIS_INSTALL_DIR}
    PATH_SUFFIXES include
    NO_DEFAULT_PATH
    REQUIRED
)

message(STATUS "Using pre-installed dependencies:")
message(STATUS "  GKlib library: ${GKLIB_LIBRARY}")
message(STATUS "  GKlib include: ${GKLIB_INCLUDE_DIR}")
message(STATUS "  METIS library: ${METIS_LIBRARY}")
message(STATUS "  METIS include: ${METIS_INCLUDE_DIR}")

# ============================================================================
# ParMETIS - Parallel graph partitioning library
# ============================================================================
file(GLOB PARMETIS_SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/libparmetis/*.c)

if(NOT PARMETIS_SOURCES)
    message(FATAL_ERROR "No ParMETIS source files found in: ${CMAKE_CURRENT_SOURCE_DIR}/libparmetis")
endif()

add_library(parmetis STATIC ${PARMETIS_SOURCES})

# Include directories
# Note: parmetislib.h includes <GKlib.h>, so GKlib must be in include path
target_include_directories(parmetis 
    PUBLIC
        ${CMAKE_CURRENT_SOURCE_DIR}/include
        ${METIS_INCLUDE_DIR}
        ${GKLIB_INCLUDE_DIR}
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/libparmetis
)

# Link libraries
target_link_libraries(parmetis 
    PUBLIC
        MPI::MPI_C
    PRIVATE
        ${METIS_LIBRARY}
        ${GKLIB_LIBRARY}
)

# Compile definitions
target_compile_definitions(parmetis PRIVATE USE_GKREGEX)

# Set integer and real type widths for ParMETIS
if(PARMETIS_USE_INT64)
    target_compile_definitions(parmetis PUBLIC IDXTYPEWIDTH=64)
else()
    target_compile_definitions(parmetis PUBLIC IDXTYPEWIDTH=32)
endif()

if(PARMETIS_USE_DOUBLE)
    target_compile_definitions(parmetis PUBLIC REALTYPEWIDTH=64)
else()
    target_compile_definitions(parmetis PUBLIC REALTYPEWIDTH=32)
endif()

# ============================================================================
# Installation
# ============================================================================
include(GNUInstallDirs)

# Install library
install(TARGETS parmetis
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

# Install header
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/include/parmetis.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

# ============================================================================
# Summary
# ============================================================================
message(STATUS "ParMETIS configuration:")
message(STATUS "  Integer width: ${PARMETIS_USE_INT64}")
message(STATUS "  Double precision: ${PARMETIS_USE_DOUBLE}")
message(STATUS "  MPI C compiler: ${MPI_C_COMPILER}")
message(STATUS "  Install prefix: ${CMAKE_INSTALL_PREFIX}")


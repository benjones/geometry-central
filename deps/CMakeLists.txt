cmake_minimum_required(VERSION 3.8)

if(POLICY CMP0169)
    cmake_policy(SET CMP0169 OLD) # use deprecated FetchContent_Populate
endif()

# We will accumulate all depenency includes and link libraries here
# (can be relative paths when building list, we will convert all to absolute at end)
SET(GC_DEP_LIBS "")

# Make sure we don't try to compile anything in the dependencies with Werror on
if ("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang" OR "${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-error")
endif()

# === Resolve Eigen dependency

message(STATUS "-- resolving Eigen 5 dependency:")

# Eigen 5 retains the historical Eigen3 package and target names. Its
# Eigen3ConfigVersion.cmake interprets this as >= 5.0 and < 6.0.
find_package(Eigen3 5 REQUIRED NO_MODULE)

if(NOT TARGET Eigen3::Eigen)
  message(FATAL_ERROR
    "Eigen 5 was found, but it did not export the Eigen3::Eigen target.")
endif()

# Imported targets are scoped to the directory which called find_package().
# Geometry-central consumes Eigen from sibling directories, so expose the
# upstream target with the same visibility as the old local interface target.
get_property(_eigen3_is_imported TARGET Eigen3::Eigen PROPERTY IMPORTED)
if(_eigen3_is_imported)
  set_property(TARGET Eigen3::Eigen PROPERTY IMPORTED_GLOBAL TRUE)
endif()

list(APPEND GC_DEP_LIBS Eigen3::Eigen)

# === done finding Eigen


# ===  Try to find suitesparse

# Look for suitesparse (adapted from ceres build system)
option(SUITESPARSE "Enable SuiteSparse." ON)
if (SUITESPARSE)
  message("-- Searching for SuiteSparse.")

  # Check for SuiteSparse and dependencies.
  find_package(SuiteSparse)
  if (SUITESPARSE_FOUND)
    # By default, if all of SuiteSparse's dependencies are found, GC is
    # built with SuiteSparse support.
    message("-- Found SuiteSparse ${SUITESPARSE_VERSION}, building with SuiteSparse.")
  else (SUITESPARSE_FOUND)
    # Disable use of SuiteSparse if it cannot be found and continue.
    message("-- Did not find all SuiteSparse dependencies, disabling SuiteSparse support.")
    update_cache_variable(SUITESPARSE OFF)
  endif (SUITESPARSE_FOUND)
else (SUITESPARSE)
  message("-- Building without SuiteSparse.")
endif (SUITESPARSE)

# HACK BY NICK
# The SuiteSparse logic above doesn't look for UMFpack, but we need it.
# This code attempts to find if by assuming that it will be in the same place
# as cholmod
if(SUITESPARSE AND SUITESPARSE_FOUND)
    string(REGEX REPLACE "cholmod" "umfpack" UMFPACK_LIBRARY ${CHOLMOD_LIBRARY})
    message("-- Guesstimated umfpack location as: ${UMFPACK_LIBRARY}")
    if(EXISTS ${UMFPACK_LIBRARY})
        list(APPEND SUITESPARSE_LIBRARIES ${UMFPACK_LIBRARY})
    else()
        message(WARNING "UMFPack guess failed, so we don't actually have SUITESPARSE support.")
        set(SUITESPARSE_FOUND FALSE)
    endif()
endif()

if(SUITESPARSE AND SUITESPARSE_FOUND)
    SET(GC_HAVE_SUITESPARSE TRUE)
    SET(GC_HAVE_SUITESPARSE TRUE PARENT_SCOPE)
    set(SUITESPARSE_INCLUDE_DIRS "${SUITESPARSE_INCLUDE_DIRS}" PARENT_SCOPE)
    set(SUITESPARSE_LIBRARIES    "${SUITESPARSE_LIBRARIES}"    PARENT_SCOPE)
else()
    SET(GC_HAVE_SUITESPARSE FALSE)
    SET(GC_HAVE_SUITESPARSE FALSE PARENT_SCOPE)
endif()

# ===  done finding Suitesparse

if (NOT TARGET nanort)
    add_library(nanort INTERFACE)
    target_include_directories(nanort INTERFACE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/nanort/include>)
endif()

if (NOT TARGET nanoflann)
    add_library(nanoflann INTERFACE)
    target_include_directories(nanoflann INTERFACE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/nanoflann/include>)
endif()

if (NOT TARGET happly)
    add_library(happly INTERFACE)
    target_include_directories(happly INTERFACE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/happly>)
endif()

# Find other simpler dependencies
list(APPEND GC_DEP_LIBS nanort)
list(APPEND GC_DEP_LIBS nanoflann)
list(APPEND GC_DEP_LIBS happly)

# == Ensure all paths are absolute

# # Helper function
# macro(convert_paths_to_absolute files)
#   foreach(path ${${files}})
#     get_filename_component(abs_path "${path}" ABSOLUTE)
#     list(REMOVE_ITEM ${files} "${path}")
#     list(APPEND ${files} "${abs_path}")
#   endforeach()
# endmacro()

# convert_paths_to_absolute(GC_DEP_LIBS)
# convert_paths_to_absolute(GC_DEP_LIBRARIES)

# Propagate dep variables up to parent
SET(GC_DEP_LIBS ${GC_DEP_LIBS} PARENT_SCOPE)
# SET(GC_DEP_LIBRARIES ${GC_DEP_LIBRARIES} PARENT_SCOPE)

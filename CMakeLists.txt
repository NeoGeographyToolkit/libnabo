cmake_minimum_required(VERSION 3.15)

set(LIB_NAME nabo)
project("lib${LIB_NAME}")

# TODO(oalexan1): These need revisiting.
set(CMAKE_CXX_STANDARD 11)

set(CMAKE_BUILD_TYPE "Release")
add_definitions(-O3)

# Extract version from header
set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR})
execute_process(
	COMMAND grep "NABO_VERSION " nabo/nabo.h
	WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
	RESULT_VARIABLE GREP_VERSION_RESULT
	OUTPUT_VARIABLE PROJECT_VERSION
	OUTPUT_STRIP_TRAILING_WHITESPACE
)
if (NOT GREP_VERSION_RESULT EQUAL 0)
	message(SEND_ERROR "Cannot grep version number: ${GREP_VERSION_RESULT}")
endif (NOT GREP_VERSION_RESULT EQUAL 0)
string(REGEX REPLACE ".*\"(.*)\".*" "\\1" PROJECT_VERSION "${PROJECT_VERSION}" )

# Documentation
#set(DOXYFILE_LATEX false)
#include(UseDoxygen)

# Compilation
add_definitions(-Wall)

# Boost required for any and cstdint on MSVC < 2010
find_package(Boost COMPONENTS REQUIRED)
if(MSVC AND (MSVC_VERSION LESS 1600))
	add_definitions(-DBOOST_STDINT)
endif(MSVC AND (MSVC_VERSION LESS 1600))

# openmp
find_package(OpenMP)
if (OPENMP_FOUND)
  add_definitions(-fopenmp -DHAVE_OPENMP)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
  if (CMAKE_COMPILER_IS_GNUCC)
    set(EXTRA_LIBS ${EXTRA_LIBS} gomp)
  endif(CMAKE_COMPILER_IS_GNUCC)
endif(OPENMP_FOUND)

# eigen 2 or 3
find_path(EIGEN_INCLUDE_DIR Eigen/Core
	/usr/local/include/eigen3
	/usr/local/include/eigen2
	/usr/local/include/eigen
	/usr/include/eigen3
	/usr/include/eigen2
	/usr/include/eigen
	/opt/local/include/eigen3
)

# optionally, opencl
# OpenCL disabled as its code is not up-to-date with API
# set(USE_OPEN_CL "false" CACHE BOOL "Set to ON to look for OpenCL")
# if (USE_OPEN_CL)
# 	find_path(OPENCL_INCLUDE_DIR CL/cl.h
# 		/usr/local/include
# 		/usr/include
# 	)
# 	if (WIN32)
# 		find_library(OPENCL_LIBRARIES opencl64)
# 		if (!OPENCL_LIBRARIES)
# 			find_library(OPENCL_LIBRARIES opencl32)
# 		endif (!OPENCL_LIBRARIES)
# 	else (WIN32)
# 		find_library(OPENCL_LIBRARIES OpenCL ENV LD_LIBRARY_PATH)
# 	endif (WIN32)
# 	if (OPENCL_INCLUDE_DIR AND OPENCL_LIBRARIES)
# 		add_definitions(-DHAVE_OPENCL)
# 		set(EXTRA_LIBS ${OPENCL_LIBRARIES} ${EXTRA_LIBS})
# 		include_directories(${OPENCL_INCLUDE_DIR})
# 		add_definitions(-DOPENCL_SOURCE_DIR=\"${CMAKE_SOURCE_DIR}/nabo/opencl/\")
# 		message("OpenCL enabled and found, enabling CL support")
# 	else (OPENCL_INCLUDE_DIR AND OPENCL_LIBRARIES)
# 		message("OpenCL enabled but not found, disabling CL support")
# 	endif (OPENCL_INCLUDE_DIR AND OPENCL_LIBRARIES)
# else(USE_OPEN_CL)
# 	message("OpenCL disabled, not looking for it")
# endif(USE_OPEN_CL)

# include all libs so far
include_directories(${CMAKE_SOURCE_DIR} ${EIGEN_INCLUDE_DIR} ${Boost_INCLUDE_DIRS})

# main nabo lib
set(NABO_SRC
	nabo/nabo.cpp
	nabo/brute_force_cpu.cpp
	nabo/kdtree_cpu.cpp
	nabo/kdtree_opencl.cpp
)
set(SHARED_LIBS "false" CACHE BOOL "To build shared (true) or static (false) library")
if (SHARED_LIBS)
	add_library(${LIB_NAME} SHARED ${NABO_SRC})
	install(TARGETS ${LIB_NAME} LIBRARY DESTINATION lib)
else(SHARED_LIBS)
	add_library(${LIB_NAME} ${NABO_SRC})
	add_definitions(-fPIC)
	install(TARGETS ${LIB_NAME} ARCHIVE DESTINATION lib)
endif(SHARED_LIBS)
set_target_properties(${LIB_NAME} PROPERTIES VERSION "${PROJECT_VERSION}" SOVERSION 1)

export(TARGETS ${LIB_NAME}
  FILE "${PROJECT_BINARY_DIR}/libnaboTargets.cmake")

install(FILES nabo/nabo.h DESTINATION include/nabo)
install(FILES README.md DESTINATION share/doc/${PROJECT_NAME})

# # create doc before installing
# set(DOC_INSTALL_TARGET "share/doc/${PROJECT_NAME}/api" CACHE STRING "Target where to install doxygen documentation")
# add_dependencies(${LIB_NAME} doc)

# if (DOXYGEN_FOUND)
# 	install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/doc/html DESTINATION ${DOC_INSTALL_TARGET})
# endif(DOXYGEN_FOUND)

# enable_testing()

#add_subdirectory(examples)
#add_subdirectory(tests)
#add_subdirectory(python)

# # Install catkin package.xml
# install(FILES package.xml DESTINATION share/libnabo)

#=============================================
# to allow find_package() on libnabo
#=============================================
# 
# the following case be used in an external project requiring libnabo:
#  ...
#  find_package(libnabo) 
#  include_directories(${libnabo_INCLUDE_DIRS}) 
#  target_link_libraries(executableName ${libnabo_LIBRARIES})
#  ...

# NOTE: the following will support find_package for 1) local build (make) and 2) for installed files (make install)

# 1- local build #

# # Register the local build in case one doesn't use "make install"
# export(PACKAGE libnabo)

# # Create variable with the library location
# get_property(libnabo_library TARGET ${LIB_NAME} PROPERTY LOCATION)

# # Create variable for the local build tree
# get_property(libnabo_include_dirs DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY INCLUDE_DIRECTORIES)
# # Configure config file for local build tree
# configure_file(libnaboConfig.cmake.in
#   "${PROJECT_BINARY_DIR}/libnaboConfig.cmake" @ONLY)

# # 2- installation build #

# # Change the library location for an install location
# get_filename_component(NABO_LIB_NAME ${libnabo_library} NAME)
# set(libnabo_library ${CMAKE_INSTALL_PREFIX}/lib/${NABO_LIB_NAME})

# # Change the include location for the case of an install location
# set(libnabo_include_dirs ${CMAKE_INSTALL_PREFIX}/include)

# # We put the generated file for installation in a different repository (i.e., ./CMakeFiles/)
# configure_file(libnaboConfig.cmake.in
#   "${PROJECT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/libnaboConfig.cmake" @ONLY)

# # The same versioning file can be used for both cases
# configure_file(libnaboConfigVersion.cmake.in
#   "${PROJECT_BINARY_DIR}/libnaboConfigVersion.cmake" @ONLY)

# install(FILES
#   "${PROJECT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/libnaboConfig.cmake"
#   "${PROJECT_BINARY_DIR}/libnaboConfigVersion.cmake"
#   "${PROJECT_BINARY_DIR}/libnaboTargets.cmake"
#   DESTINATION share/libnabo/cmake COMPONENT dev)

# #=============================================
# # Add uninstall target
# #=============================================
# configure_file(
#     "${CMAKE_CURRENT_SOURCE_DIR}/cmake_uninstall.cmake.in"
#     "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
#     IMMEDIATE @ONLY)

# add_custom_target(uninstall
#     COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)

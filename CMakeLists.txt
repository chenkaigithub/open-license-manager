#TODO guess address_model 64 in boost
#make it compatible with find_boost (exchange lowercase to camelcase)
#fix shared libraries activation

#cmake_minimum_required(VERSION 3.0.0 FATAL_ERROR)
cmake_minimum_required(VERSION 2.8.11 FATAL_ERROR)

#version variables, major and minor must be 1 character only
SET(LICENSECC_MAJOR_VERSION 1)
SET(LICENSECC_MINOR_VERSION 0)
SET(LICENSECC_PATCH_VERSION 0 CACHE STRING "License++ patch version string")
SET(LICENSECC_INT_VERSION "${LICENSECC_MAJOR_VERSION}${LICENSECC_MINOR_VERSION}${LICENSECC_PATCH_VERSION}")
SET(LICENSECC_VERSION "${LICENSECC_MAJOR_VERSION}.${LICENSECC_MINOR_VERSION}.${LICENSECC_PATCH_VERSION}")
SET(LICENSECC_SHORT_LICENSE "BSD Software License")

SET(CMAKE_DISABLE_SOURCE_CHANGES ON)
SET(CMAKE_DISABLE_IN_SOURCE_BUILD ON)
SET(CMAKE_VERBOSE_MAKEFILE ON CACHE BOOL "CMake verbose" FORCE)

project (license++ C CXX)
SET(LIBRARY_OUTPUT_PATH ${PROJECT_BINARY_DIR}/lib)
SET(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/build/modules/")

include(utilities)
if(CMAKE_BUILD_TYPE)
	string(TOLOWER ${CMAKE_BUILD_TYPE} _CMAKE_BUILD_TYPE)
	EVAL_CONDITION(RELEASE_BUILD ${_CMAKE_BUILD_TYPE} STREQUAL "release")
else(CMAKE_BUILD_TYPE)
	set(RELEASE_BUILD 0)
endif(CMAKE_BUILD_TYPE)

if(WIN32)

	SET(PLATFORM_LIBS "")
	include(target_arch)
	target_architecture( TARGET_ARCHITECTURE )
	message(STATUS "architecture detected: ${TARGET_ARCHITECTURE}")
	#force to patch an error in cmake
	if(TARGET_ARCHITECTURE STREQUAL "x86_64")
		SET(CMAKE_EXE_LINKER_FLAGS "/machine:X64" CACHE STRING "Linker flags" FORCE)
		SET(CMAKE_MODULE_LINKER_FLAGS "/machine:X64" CACHE STRING "Module Linker flags" FORCE)
		SET(CMAKE_SHARED_LINKER_FLAGS "/machine:X64" CACHE STRING "Shared Linker flags" FORCE)
	endif()
	SET(CMAKE_FIND_LIBRARY_SUFFIXES .lib ) #${CMAKE_FIND_LIBRARY_SUFFIXES}
	add_definitions("/D_CRT_SECURE_NO_WARNINGS")
else(WIN32)
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pipe -fmessage-length=0 -std=c++11 -Wuninitialized -fPIC") #-Wall
    SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -pipe -fmessage-length=0 -Wall -Wuninitialized -fPIC") 
    #use static libraries in release to make it easier to distribute it
    #SET(Boost_USE_STATIC_RUNTIME ${RELEASE_BUILD})
    if(RELEASE_BUILD)
        SET(CMAKE_FIND_LIBRARY_SUFFIXES .a) #.so for dynamic libraries
    else()
        SET(CMAKE_FIND_LIBRARY_SUFFIXES .a .so) 
    endif()
    set (CMAKE_EXE_LINKER_FLAGS  "-pthread")
    SET(CMAKE_EXE_LINKER_FLAGS_RELEASE "-Wl,--strip-all -static-libstdc++") #-static 
    #you need to "force" the change in cmake_install_prefix after the project keyword
	IF(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
		SET(CMAKE_INSTALL_PREFIX "/usr/local" CACHE PATH "Install path prefix" FORCE)
	ENDIF(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
endif(WIN32)
set_property(DIRECTORY APPEND PROPERTY COMPILE_DEFINITIONS $<$<CONFIG:Debug>:_DEBUG>)


if(CMAKE_SIZEOF_VOID_P EQUAL 8)
  SET(HAVE_64BIT_SIZE_T 1)
else(CMAKE_SIZEOF_VOID_P EQUAL 8)
  SET(HAVE_64BIT_SIZE_T 0)
endif(CMAKE_SIZEOF_VOID_P EQUAL 8)

include(add_boost)
add_boost(STATIC MODULES date_time test program_options system filesystem) 
#set below in case of dynamic linking in debug.
#set_property(DIRECTORY APPEND PROPERTY COMPILE_DEFINITIONS $<$<CONFIG:Debug>:BOOST_TEST_DYN_LINK>)
add_definitions(-DBOOST_ALL_NO_LIB) #Disable Boost Microsoft magic, all dependencies are handled by cmake
add_definitions(-DBOOST_LIB_DIAGNOSTIC) #Check it is really disabled

if(NOT WIN32)	
	#find a static version of openssl crypto library
	find_package(OpenSSL REQUIRED)
	include_directories(${OPENSSL_INCLUDE_DIR})
	list(APPEND EXTERNAL_LIBS ${OPENSSL_CRYPTO_LIBRARY})
	MESSAGE(STATUS "Found openssl version ${OPENSSL_VERSION}")
	
	#Zlib required when openssl version < 1.0.1f
	find_package(ZLIB REQUIRED)
	list(APPEND EXTERNAL_LIBS ${ZLIB_LIBRARIES})
	
	if(USE_DBUS_IDENTIFIER)
	    FIND_PACKAGE(Dbus REQUIRED)
	    add_definitions(-DUSE_DBUS)
	    include_directories(${DBUS_INCLUDE_DIR})
	    include_directories(${DBUS_ARCH_INCLUDE_DIR})
	    list(APPEND EXTERNAL_LIBS ${DBUS_LIBRARIES})
	endif(USE_DBUS_IDENTIFIER)    
	list(APPEND EXTERNAL_LIBS dl)
endif(NOT WIN32)

FIND_PACKAGE(Doxygen)

IF (DOXYGEN_FOUND)

	SET(DOXYGEN_INPUT ${CMAKE_BINARY_DIR}/doc/Doxyfile)
	SET(DOXYGEN_OUTPUT ${CMAKE_BINARY_DIR}/doc/doxy_output)
	
	configure_file (
	  "doc/Doxyfile.in"
	  "doc/Doxyfile"
	)
	
	ADD_CUSTOM_COMMAND(
	  OUTPUT ${DOXYGEN_OUTPUT}
	  COMMAND ${CMAKE_COMMAND} -E echo_append "Building API Documentation..."
	  COMMAND ${DOXYGEN_EXECUTABLE} ${DOXYGEN_INPUT}
	  COMMAND ${CMAKE_COMMAND} -E echo "Done."
	  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
	  DEPENDS ${DOXYGEN_INPUT})
	
	ADD_CUSTOM_TARGET(apidoc ALL DEPENDS ${DOXYGEN_OUTPUT})
	
	ADD_CUSTOM_TARGET(apidoc_forced
	  COMMAND ${CMAKE_COMMAND} -E echo_append "Building API Documentation..."
	  COMMAND ${DOXYGEN_EXECUTABLE} ${DOXYGEN_INPUT}
	  COMMAND ${CMAKE_COMMAND} -E echo "Done."
	  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})
ENDIF (DOXYGEN_FOUND)

configure_file (
	"src/build_properties.h.in" 
	"${CMAKE_BINARY_DIR}/build_properties.h"
)
#include build directory to find build_properties.h
include_directories(${CMAKE_BINARY_DIR})

add_subdirectory(src)

INCLUDE(CTest)
ENABLE_TESTING()
IF(BUILD_TESTING)
  SET(BUILDNAME "${BUILDNAME}" CACHE STRING "Name of build on the dashboard")
  MARK_AS_ADVANCED(BUILDNAME)
ENDIF(BUILD_TESTING)
add_subdirectory(test)

#include(build/modules/CmakeDebugVariables.txt)
message( STATUS "C compiler: " ${CMAKE_C_COMPILER})
message( STATUS "C compiler flags: " ${CMAKE_C_FLAGS})
message( STATUS "CXX compiler: " ${CMAKE_CXX_COMPILER})
message( STATUS "CXX compiler flags: " ${CMAKE_CXX_FLAGS})
if(CMAKE_BUILD_TYPE)
  message( STATUS "Build type: " ${CMAKE_BUILD_TYPE})
endif(CMAKE_BUILD_TYPE)
message( STATUS "Install prefix: " ${CMAKE_INSTALL_PREFIX})
if(DOXYGEN_FOUND)
  message(STATUS "Doxygen: " ${DOXYGEN_EXECUTABLE})
else(DOXYGEN_FOUND)
  message(STATUS "Doxygen: not found.")
endif(DOXYGEN_FOUND)

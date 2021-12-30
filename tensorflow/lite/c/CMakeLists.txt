#
# Copyright 2021 The TensorFlow Authors. All Rights Reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      https://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

cmake_minimum_required(VERSION 3.16)
project(tensorflow-lite-c C CXX)
set (PROJECT_VERSION_MAJOR 2)
set (PROJECT_VERSION_MINOR 4)
set (PROJECT_VERSION_PATCH 3)
set (PROJECT_VERSION ${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}.${PROJECT_VERSION_PATCH})
set (TENSORFLOW_LITE_C_NAME tensorflowlite_c)
set (TENSORFLOW_LITE_C_NAME_UPPER TENSORFLOWLITE_C )

include(GNUInstallDirs)

option(TFLITE_C_BUILD_SHARED_LIBS "Build shared libraries" ON)
set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)

if(TFLITE_C_BUILD_SHARED_LIBS)
    set( TENSORFLOW_LITE_C_SUFFIX ${CMAKE_SHARED_LIBRARY_SUFFIX})
    set( TENSORFLOW_LITE_C_PREFIX ${CMAKE_SHARED_LIBRARY_PREFIX})
else()
    set( TENSORFLOW_LITE_C_SUFFIX ${CMAKE_STATIC_LIBRARY_SUFFIX})
    set( TENSORFLOW_LITE_C_PREFIX ${CMAKE_STATIC_LIBRARY_PREFIX})
endif()

set(TENSORFLOW_SOURCE_DIR "" CACHE PATH
  "Directory that contains the TensorFlow project"
)
if (NOT TENSORFLOW_SOURCE_DIR)
  get_filename_component(TENSORFLOW_SOURCE_DIR
    "${CMAKE_CURRENT_LIST_DIR}/../../../"
    ABSOLUTE
  )
endif()

file(GLOB_RECURSE TFL_INC_HEADERS "${TENSORFLOW_SOURCE_DIR}/tensorflow/lite/*.h")

add_subdirectory(
  "${TENSORFLOW_SOURCE_DIR}/tensorflow/lite"
  "${CMAKE_CURRENT_BINARY_DIR}/tensorflow-lite"
  EXCLUDE_FROM_ALL
)

set(CMAKE_CXX_STANDARD 11)

if(CMAKE_SYSTEM_NAME MATCHES "Windows"
    AND (MSVC AND (CMAKE_SIZEOF_VOID_P EQUAL 4)))
  message("Disabling MSVC /O2 optimization for Win32")
  set(CompFlags
    CMAKE_CXX_FLAGS_RELEASE
    CMAKE_CXX_FLAGS_MINSIZEREL
    CMAKE_CXX_FLAGS_RELWITHDEBINFO
    CMAKE_C_FLAGS_RELEASE
    CMAKE_C_FLAGS_MINSIZEREL
    CMAKE_C_FLAGS_RELWITHDEBINFO
  )
  foreach (CompFlag ${CompFlags})
    string(REGEX REPLACE "(\/Ob. )" "" ${CompFlag} "${${CompFlag}}")
    string(REPLACE "/O2" "/O1" ${CompFlag} "${${CompFlag}}")
    list(REMOVE_DUPLICATES ${CompFlag})
    set(${CompFlag} "${${CompFlag}}" CACHE INTERNAL "")
  endforeach()
endif()

set(TFLITE_C_LIBTYPE STATIC)
if (TFLITE_C_BUILD_SHARED_LIBS)
  set(TFLITE_C_LIBTYPE SHARED)
endif()

add_library(tensorflowlite_c ${TFLITE_C_LIBTYPE}
  builtin_op_data.h
  common.h
  common.c
  c_api_types.h
  c_api.h
  c_api.cc
  c_api_experimental.h
  c_api_experimental.cc
  c_api_internal.h
)

if (TFLITE_C_BUILD_SHARED_LIBS)
  if (WIN32)
    target_compile_definitions(tensorflowlite_c PRIVATE TFL_COMPILE_LIBRARY)
  elseif (APPLE)
    target_link_options(tensorflowlite_c PRIVATE "-Wl,-exported_symbols_list,${TENSORFLOW_SOURCE_DIR}/tensorflow/lite/c/exported_symbols.lds")
  else ()
    target_link_options(tensorflowlite_c PRIVATE "-Wl,--version-script,${TENSORFLOW_SOURCE_DIR}/tensorflow/lite/c/version_script.lds")
  endif()
endif()

target_link_libraries(tensorflowlite_c
  tensorflow-lite
)

install( TARGETS tensorflowlite_c 
   RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
   ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

foreach ( file ${TFL_INC_HEADERS} )
    get_filename_component( dir ${file} DIRECTORY )
    string(REGEX REPLACE "${TENSORFLOW_SOURCE_DIR}/" "" dir_ ${dir})
    install( FILES ${file} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${dir_} )
    # message(STATUS "Installing ${file} into ${CMAKE_INSTALL_INCLUDEDIR}/${dir_}")
endforeach()

configure_file (project-config.cmake.in ${PROJECT_NAME}-config.cmake @ONLY)
configure_file (project-config-version.cmake.in
  ${PROJECT_NAME}-config-version.cmake @ONLY)
  
install (FILES
  "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config.cmake"
  DESTINATION "${CMAKE_INSTALL_PREFIX}")
  
install (FILES
  "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config-version.cmake"
  DESTINATION "${CMAKE_INSTALL_PREFIX}")


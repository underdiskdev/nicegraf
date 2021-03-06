#[[
Copyright © 2020 nicegraf contributors

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the “Software”), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of
the Software, and to permit persons to whom the Software is furnished to do so,
subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED “AS IS”, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
]]

cmake_minimum_required(VERSION 3.14.0)
project(nicegraf)

# This function adds a new library target and sets some configuration
# common across all libs.
function (ngf_lib_target)
  cmake_parse_arguments(LIB "" "NAME" "SRCS;DEPS;COPTS" ${ARGN})
  message(WARNING "lib name is ${LIB_NAME}")
  add_library(${LIB_NAME} STATIC ${LIB_SRCS})
  if ( LIB_DEPS )
    target_link_libraries(${LIB_NAME} ${LIB_DEPS})
  endif()
  target_include_directories(${LIB_NAME}
                             PUBLIC ${CMAKE_CURRENT_LIST_DIR}/include
                             PRIVATE ${CMAKE_CURRENT_LIST_DIR}/source)
  if(WIN32)
    set (NICEGRAF_COMMON_COMPILE_OPTS "/W4" "/WX")
  else()
    set(NICEGRAF_COMMON_COMPILE_OPTS "-Wall" "-Wconversion" "-Wno-unknown-pragmas")
  endif()
  target_compile_options(${LIB_NAME} PRIVATE ${NICEGRAF_COMMON_COMPILE_OPTS})
  if ( LIB_COPTS )
    target_compile_options(${LIB_NAME} PRIVATE ${LIB_COPTS})
  endif()
endfunction()


# A helper library for parsing shader metadata produced by
# nicegraf-shaderc.
ngf_lib_target(NAME ngf_metadata_parser
               SRCS ${CMAKE_CURRENT_LIST_DIR}/include/metadata_parser.h
                    ${CMAKE_CURRENT_LIST_DIR}/source/metadata_parser.c)

# A library with various utilities shared internally across different backends.
ngf_lib_target(NAME nicegraf_internal
               SRCS ${CMAKE_CURRENT_LIST_DIR}/include/nicegraf.h
                    ${CMAKE_CURRENT_LIST_DIR}/source/nicegraf_internal.h
                    ${CMAKE_CURRENT_LIST_DIR}/source/nicegraf_internal.c
                    ${CMAKE_CURRENT_LIST_DIR}/source/stack_alloc.c
               DEPS ngf_metadata_parser)

# nicegraf utility library.
ngf_lib_target(NAME nicegraf_util
           SRCS ${CMAKE_CURRENT_LIST_DIR}/include/nicegraf_util.h
                ${CMAKE_CURRENT_LIST_DIR}/source/nicegraf_util.c
           DEPS nicegraf_internal)

# Common dependencies used by nicegraf on all platforms.
set(NICEGRAF_COMMON_DEPS nicegraf_internal ngf_metadata_parser)

if (APPLE)
  # Find apple-specific system libs.
  find_library(APPLE_METAL Metal)
  find_library(APPLE_QUARTZ_CORE QuartzCore)
  if(${CMAKE_SYSTEM_NAME} STREQUAL "iOS")
    find_library(APPLE_KIT UIKit)
  else()
    find_library(APPLE_KIT AppKit)
  endif()

  # Nicegraf with Metal backend.
  ngf_lib_target(NAME nicegraf_mtl
                 SRCS ${CMAKE_CURRENT_LIST_DIR}/include/nicegraf.h
                      ${CMAKE_CURRENT_LIST_DIR}/source/nicegraf_impl_metal.mm
                 DEPS ${NICEGRAF_COMMON_DEPS} ${APPLE_METAL} ${APPLE_QUARTZ_CORE} ${APPLE_KIT}
                 COPTS "--std=c++17" "-fobjc-arc")
else()
  # Import the EGL library for context creation.
  set(EGL_NO_GLEW ON)
  add_subdirectory(${CMAKE_CURRENT_LIST_DIR}/third_party/EGL/EGL)

  # Nicegraf with GL backend.
  ngf_lib_target(NAME nicegraf_gl
                 SRCS ${CMAKE_CURRENT_LIST_DIR}/include/nicegraf.h
                      ${CMAKE_CURRENT_LIST_DIR}/source/nicegraf_impl_gl43.c
                      ${CMAKE_CURRENT_LIST_DIR}/source/gl_43_core.c
                 DEPS egl)

  # Try searching for Vulkan SDK.
  find_package(Vulkan)

  # Only build the Vulkan backend if the Vulkan SDK is present.
  if ( Vulkan_FOUND )
    # Import VMA for handling vulkan memory allocation.
    add_definitions("-DVMA_STATIC_VULKAN_FUNCTIONS=0")
    add_subdirectory(${CMAKE_CURRENT_LIST_DIR}/third_party/vma)

    # Some vulkan-specific dependencies.
    set(NICEGRAF_VK_DEPS vma)
    if (NOT WIN32)
      set(NICEGRAF_VK_DEPS ${NICEGRAF_VK_DEPS} xcb)
    endif()

    # Vulkan backend.
    ngf_lib_target(NAME nicegraf_vk
                   SRCS ${CMAKE_CURRENT_LIST_DIR}/include/nicegraf.h
                        ${CMAKE_CURRENT_LIST_DIR}/source/nicegraf_impl_vk.c
                        ${CMAKE_CURRENT_LIST_DIR}/source/vk_10.c
                   DEPS ${NICEGRAF_COMMON_DEPS} ${NICEGRAF_VK_DEPS})

    # Add Vulkan header directory to nicegraf_vk include paths.
    get_property(VK_INCLUDE_PATH TARGET Vulkan::Vulkan PROPERTY INTERFACE_INCLUDE_DIRECTORIES)
    target_include_directories(nicegraf_vk PRIVATE ${VK_INCLUDE_PATH})
  else()
    message(WARNING "nicegraf Vulkan backend will not be built because Vulkan SDK was not found.")
  endif()
endif()

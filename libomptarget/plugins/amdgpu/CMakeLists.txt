##===----------------------------------------------------------------------===##
#
#                     The LLVM Compiler Infrastructure
#
# This file is dual licensed under the MIT and the University of Illinois Open
# Source Licenses. See LICENSE.txt for details.
#
##===----------------------------------------------------------------------===##
#
# Build a plugin for an AMDGPU machine if available.
#
##===----------------------------------------------------------------------===##

################################################################################

# as of rocm-3.7, hsa is installed with cmake packages and kmt is found via hsa
find_package(hsa-runtime64 QUIET 1.2.0 HINTS ${CMAKE_INSTALL_PREFIX} PATHS /opt/rocm)
if (NOT ${hsa-runtime64_FOUND})
  libomptarget_say("Not building AMDGPU plugin: hsa-runtime64 not found")
  return()
endif()

if(NOT LIBOMPTARGET_DEP_LIBELF_FOUND)
  libomptarget_say("Not building AMDGPU plugin: LIBELF not found")
  return()
endif()

if(NOT CMAKE_SYSTEM_PROCESSOR MATCHES "(x86_64)|(ppc64le)|(aarch64)$" AND CMAKE_SYSTEM_NAME MATCHES "Linux")
  libomptarget_say("Not building AMDGPU plugin: only support AMDGPU in Linux x86_64, ppc64le, or aarch64 hosts.")
  return()
endif()

if (NOT LIBOMPTARGET_LLVM_INCLUDE_DIRS)
  libomptarget_say("Not building AMDGPU plugin: Missing definition for LIBOMPTARGET_LLVM_INCLUDE_DIRS")
  return()
endif()

libomptarget_say("Building amdgpu offloading plugin")

################################################################################
# Define the suffix for the runtime messaging dumps.
add_definitions(-DTARGET_NAME=AMDGPU)
if(CMAKE_SYSTEM_PROCESSOR MATCHES "(ppc64le)|(aarch64)$")
   add_definitions(-DLITTLEENDIAN_CPU=1)
endif()

if(CMAKE_BUILD_TYPE MATCHES Debug)
  add_definitions(-DDEBUG)
endif()

include_directories(
  ${CMAKE_CURRENT_SOURCE_DIR}/impl
  ${LIBOMPTARGET_LLVM_INCLUDE_DIRS}
)

add_library(bolt-omptarget.rtl.amdgpu SHARED
      impl/atmi.cpp
      impl/atmi_interop_hsa.cpp
      impl/data.cpp
      impl/get_elf_mach_gfx_name.cpp
      impl/machine.cpp
      impl/system.cpp
      impl/utils.cpp
      impl/msgpack.cpp
      src/rtl.cpp
      )

# Install plugin under the lib destination folder.
# When we build for debug, OPENMP_LIBDIR_SUFFIX get set to -debug
install(TARGETS bolt-omptarget.rtl.amdgpu LIBRARY DESTINATION "lib${OPENMP_LIBDIR_SUFFIX}")

# Install aliases
get_target_property(BOLT_LIBOMPTARGET_LIBRARY_DIR bolt-omptarget.rtl.amdgpu LIBRARY_OUTPUT_DIRECTORY)
if(BOLT_LIBOMPTARGET_LIBRARY_DIR)
  add_custom_command(TARGET bolt-omptarget.rtl.amdgpu POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E create_symlink ${CMAKE_SHARED_LIBRARY_PREFIX}bolt-omptarget.rtl.amdgpu${CMAKE_SHARED_LIBRARY_SUFFIX}
      ${CMAKE_SHARED_LIBRARY_PREFIX}omptarget.rtl.amdgpu${CMAKE_SHARED_LIBRARY_SUFFIX}
      WORKING_DIRECTORY ${BOLT_LIBOMPTARGET_LIBRARY_DIR}
  )
endif()
install(CODE "execute_process(COMMAND \"\${CMAKE_COMMAND}\" -E create_symlink \"${CMAKE_SHARED_LIBRARY_PREFIX}bolt-omptarget.rtl.amdgpu${CMAKE_SHARED_LIBRARY_SUFFIX}\"
  \"${CMAKE_SHARED_LIBRARY_PREFIX}omptarget.rtl.amdgpu${CMAKE_SHARED_LIBRARY_SUFFIX}\" WORKING_DIRECTORY
  \$ENV{DESTDIR}\${CMAKE_INSTALL_PREFIX}/${OPENMP_INSTALL_LIBDIR})")

set_property(TARGET bolt-omptarget.rtl.amdgpu PROPERTY INSTALL_RPATH "$ORIGIN")
target_link_libraries(
  bolt-omptarget.rtl.amdgpu
  PRIVATE
  elf_common
  hsa-runtime64::hsa-runtime64
  pthread dl elf
  "-Wl,--version-script=${CMAKE_CURRENT_SOURCE_DIR}/../exports"
  "-Wl,-z,defs"
  )

# Report to the parent scope that we are building a plugin for amdgpu
set(LIBOMPTARGET_SYSTEM_TARGETS "${LIBOMPTARGET_SYSTEM_TARGETS} amdgcn-amd-amdhsa" PARENT_SCOPE)


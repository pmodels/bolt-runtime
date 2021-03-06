##===----------------------------------------------------------------------===##
#
# Build a plugin for a NEC Aurora machine if available. (Can also run on host)
#
##===----------------------------------------------------------------------===##


if(${LIBOMPTARGET_DEP_VEO_FOUND})
  libomptarget_say("Building SX-Aurora VE offloading plugin.")
  set(additional_libs "")
  set(additional_libs ${LIBOMPTARGET_DEP_VEO_LIBRARIES}
                      ${LIBOMPTARGET_DEP_VEOSINFO_LIBRARIES}
                      ${additional_libs})

  set(tmachine_name "ve")
  set(tmachine_libname "ve")
  set(tmachine_triple "ve-unknown-linux-unknown")
  set(elf_machine_id 251)

  include_directories(${LIBOMPTARGET_DEP_LIBELF_INCLUDE_DIR})
  include_directories(${LIBOMPTARGET_DEP_VEO_INCLUDE_DIR})


  # Define macro to be used as prefix of the runtime messages for this target.
  add_definitions("-DTARGET_NAME=${tmachine_name}")

  # Define macro with the ELF ID for this target.
  add_definitions("-DTARGET_ELF_ID=${elf_machine_id}")

  add_library("bolt-omptarget.rtl.${tmachine_libname}" SHARED
    ${CMAKE_CURRENT_SOURCE_DIR}/src/rtl.cpp)

  # Install plugin under the lib destination folder.
  install(TARGETS "bolt-omptarget.rtl.${tmachine_libname}"
    LIBRARY DESTINATION lib${OPENMP_LIBDIR_SUFFIX})

  # install aliases for BOLT
  add_custom_command(TARGET bolt-omptarget.rtl.${tmachine_libname} POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E create_symlink ${CMAKE_SHARED_LIBRARY_PREFIX}bolt-omptarget.rtl.${tmachine_libname}${CMAKE_SHARED_LIBRARY_SUFFIX}
    ${CMAKE_SHARED_LIBRARY_PREFIX}omptarget.rtl.${tmachine_libname}${CMAKE_SHARED_LIBRARY_SUFFIX}
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
  )
  install(CODE "execute_process(COMMAND \"\${CMAKE_COMMAND}\" -E create_symlink \"${CMAKE_SHARED_LIBRARY_PREFIX}bolt-omptarget.rtl.${tmachine_libname}${CMAKE_SHARED_LIBRARY_SUFFIX}\"
    \"${CMAKE_SHARED_LIBRARY_PREFIX}omptarget.rtl.${tmachine_libname}${CMAKE_SHARED_LIBRARY_SUFFIX}\" WORKING_DIRECTORY
    \$ENV{DESTDIR}\${CMAKE_INSTALL_PREFIX}/${OPENMP_INSTALL_LIBDIR})")

  target_link_libraries(
    "bolt-omptarget.rtl.${tmachine_libname}"
    elf_common
    ${LIBOMPTARGET_DEP_LIBFFI_LIBRARIES}
    ${LIBOMPTARGET_DEP_LIBELF_LIBRARIES}
    ${additional_libs}
    "-Wl,--version-script=${CMAKE_CURRENT_SOURCE_DIR}/../exports -Wl,-z,defs")

  # Report to the parent scope that we are building a plugin.
  set(LIBOMPTARGET_SYSTEM_TARGETS
    "${LIBOMPTARGET_SYSTEM_TARGETS} ${tmachine_triple}" PARENT_SCOPE)
else()
    libomptarget_say("Not building nec-aurora plugin: libveo or libveosinfo not found.")
endif()

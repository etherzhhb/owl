# Check if this is a in tree build.
if (NOT DEFINED LLVM_MAIN_SRC_DIR)
  project(Owl)
  cmake_minimum_required(VERSION 2.8)

  # Where is LLVM installed?
  set(LLVM_INSTALL_ROOT "" CACHE PATH "Root of LLVM install.")
  # Check if the LLVM_INSTALL_ROOT valid.
  if( NOT EXISTS ${LLVM_INSTALL_ROOT}/include/llvm )
    message(FATAL_ERROR "LLVM_INSTALL_ROOT (${LLVM_INSTALL_ROOT}) is not a valid LLVM installation.")
  endif(NOT EXISTS ${LLVM_INSTALL_ROOT}/include/llvm)
  #FileCheck is not install by default, warn the user to Copy FileCheck
  if( NOT EXISTS ${LLVM_INSTALL_ROOT}/bin/FileCheck
      OR NOT EXISTS ${LLVM_INSTALL_ROOT}/bin/not)
    message(WARNING "FileCheck or not are required by running regress tests, "
                    "but they are not installed! Please copy it to "
                    "${LLVM_INSTALL_ROOT}/bin.")
  endif(NOT EXISTS ${LLVM_INSTALL_ROOT}/bin/FileCheck
        OR NOT EXISTS ${LLVM_INSTALL_ROOT}/bin/not)
  # Add the llvm header path.
  include_directories(${LLVM_INSTALL_ROOT}/include/)

  # Get the system librarys that will link into LLVM.
  function(get_system_libs return_var)
    # Returns in `return_var' a list of system libraries used by LLVM.
    if( NOT MSVC )
      if( MINGW )
        set(system_libs ${system_libs} imagehlp psapi)
      elseif( CMAKE_HOST_UNIX )
        if( HAVE_LIBDL )
          set(system_libs ${system_libs} ${CMAKE_DL_LIBS})
        endif()
        if( LLVM_ENABLE_THREADS AND HAVE_LIBPTHREAD )
          set(system_libs ${system_libs} pthread)
        endif()
      endif( MINGW )
    endif( NOT MSVC )
    set(${return_var} ${system_libs} PARENT_SCOPE)
  endfunction(get_system_libs)

  # Now set the header paths.
  execute_process(COMMAND "${LLVM_INSTALL_ROOT}/bin/llvm-config" --includedir
                  OUTPUT_VARIABLE LLVM_INCLUDE_DIR
                  OUTPUT_STRIP_TRAILING_WHITESPACE)
  include_directories( ${LLVM_INCLUDE_DIR} )

  # Get the TARGET_TRIPLE
  execute_process(COMMAND "${LLVM_INSTALL_ROOT}/bin/llvm-config" --host-target
                  OUTPUT_VARIABLE TARGET_TRIPLE
                  OUTPUT_STRIP_TRAILING_WHITESPACE)

  # And then set the cxx flags.
  execute_process(COMMAND "${LLVM_INSTALL_ROOT}/bin/llvm-config" --cxxflags
                  OUTPUT_VARIABLE LLVM_CXX_FLAGS
                  OUTPUT_STRIP_TRAILING_WHITESPACE)
  set(CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS} ${LLVM_CXX_FLAGS})

  # Check LLVM_ENABLE_ASSERTIONS
  execute_process(COMMAND "${LLVM_INSTALL_ROOT}/bin/llvm-config" --assertion-mode
                  OUTPUT_VARIABLE LLVM_ENABLE_ASSERTIONS
                  OUTPUT_STRIP_TRAILING_WHITESPACE)
  # Copied from LLVM's HandleLLVMOptions.cmake
  if( LLVM_ENABLE_ASSERTIONS )
    # MSVC doesn't like _DEBUG on release builds. See PR 4379.
    if( NOT MSVC )
      add_definitions( -D_DEBUG )
    endif()
    # On non-Debug builds cmake automatically defines NDEBUG, so we
    # explicitly undefine it:
    if( NOT uppercase_CMAKE_BUILD_TYPE STREQUAL "DEBUG" )
      add_definitions( -UNDEBUG )
      # Also remove /D NDEBUG to avoid MSVC warnings about conflicting defines.
      foreach (flags_var_to_scrub
          CMAKE_CXX_FLAGS_RELEASE
          CMAKE_CXX_FLAGS_RELWITHDEBINFO
          CMAKE_CXX_FLAGS_MINSIZEREL
          CMAKE_C_FLAGS_RELEASE
          CMAKE_C_FLAGS_RELWITHDEBINFO
          CMAKE_C_FLAGS_MINSIZEREL)
        string (REGEX REPLACE "(^| )[/-]D *NDEBUG($| )" " "
          "${flags_var_to_scrub}" "${${flags_var_to_scrub}}")
      endforeach()
    endif()
  endif()

  # Make sure the isl c files are built as fPIC
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIC")
endif(NOT DEFINED LLVM_MAIN_SRC_DIR)

set(OWL_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR})
set(OWL_BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR})

# Add path for custom modules
set(CMAKE_MODULE_PATH
  ${CMAKE_MODULE_PATH}
  "${CMAKE_CURRENT_SOURCE_DIR}/cmake"
  )

include("owl_macros")

# Add appropriate flags for GCC
if (CMAKE_COMPILER_IS_GNUCXX)
  # FIXME: Turn off exceptions, RTTI:
  # -fno-exceptions -fno-rtti
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-common -Woverloaded-virtual -Wno-long-long -Wall -W -Wno-unused-parameter -Wwrite-strings -fno-exceptions -fno-rtti")
elseif (MSVC)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /EHs-c-")
  add_definitions("-D_HAS_EXCEPTIONS=0")
else ()
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-exceptions -fno-rtti")
endif ()

# Add path for custom modules
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${OWL_SOURCE_DIR}/cmake")

SET(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

# Support GPGPU code generation if the library is available.
if (CUDALIB_FOUND)
  INCLUDE_DIRECTORIES( ${CUDALIB_INCLUDE_DIR} )
endif(CUDALIB_FOUND)

include_directories(
  BEFORE
  ${CMAKE_CURRENT_SOURCE_DIR}/include
  ${CMAKE_CURRENT_BINARY_DIR}/include
  )

install(DIRECTORY include/
  DESTINATION include
  FILES_MATCHING
  PATTERN "*.h"
  PATTERN ".svn" EXCLUDE
  )

install(DIRECTORY ${OWL_BINARY_DIR}/include/
  DESTINATION include
  FILES_MATCHING
  PATTERN "*.h"
  PATTERN "CMakeFiles" EXCLUDE
  PATTERN ".svn" EXCLUDE
  )

add_definitions( -D_GNU_SOURCE )

#add_subdirectory(docs)
#add_subdirectory(lib)
#add_subdirectory(test)
#add_subdirectory(tools)
# TODO: docs.


#configure_file( ${CMAKE_CURRENT_SOURCE_DIR}/include/owl/Config/config.h.cmake
#                ${OWL_BINARY_DIR}/include/owl/Config/config.h )

# Add target to check formatting of owl files
file( GLOB_RECURSE files *.h *.cpp)

set(check_format_depends)
set(update_format_depends)
set(i 0)
foreach (file IN LISTS files)
  add_custom_command(OUTPUT owl-check-format${i}
    COMMAND clang-format -sort-includes -style=llvm ${file} | diff -u ${file} -
    VERBATIM
    COMMENT "Checking format of ${file}..."
  )
  list(APPEND check_format_depends "owl-check-format${i}")

  add_custom_command(OUTPUT owl-update-format${i}
    COMMAND clang-format -sort-includes -i -style=llvm ${file}
    VERBATIM
    COMMENT "Updating format of ${file}..."
  )
  list(APPEND update_format_depends "owl-update-format${i}")

  math(EXPR i ${i}+1)
endforeach ()

add_custom_target(owl-check-format DEPENDS ${check_format_depends})
set_target_properties(owl-check-format PROPERTIES FOLDER "Owl")

add_custom_target(owl-update-format DEPENDS ${update_format_depends})
set_target_properties(owl-update-format PROPERTIES FOLDER "Owl")

# Set the variable OWL_LINK_LIBS in the llvm/tools/ dir.
set(OWL_LINK_LIBS ${OWL_LINK_LIBS} PARENT_SCOPE)

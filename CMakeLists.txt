cmake_minimum_required(VERSION 3.4.3)

# If we are not building as a part of LLVM/Clang, build IWYU as a
# standalone project, using LLVM and Clang as an external libraries:
if( CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR )
  set(IWYU_IS_BUILT_STANDALONE TRUE)
  message(STATUS "IWYU: is built standalone.")
  if( POLICY CMP0048 )
    # Silence CMP0048 warning about missing project VERSION.
    cmake_policy(SET CMP0048 NEW)
  endif()

  project(include-what-you-use)

  # Use llvm-config to discover CMake module directories
  find_program(LLVM_CONFIG_EXECUTABLE "llvm-config")
  if (LLVM_CONFIG_EXECUTABLE)
    message(STATUS "IWYU: Found llvm-config as ${LLVM_CONFIG_EXECUTABLE}")
    execute_process(
      COMMAND ${LLVM_CONFIG_EXECUTABLE} --cmakedir
      RESULT_VARIABLE HAD_ERROR
      OUTPUT_VARIABLE CONFIG_OUTPUT
    )
  else(LLVM_CONFIG_EXECUTABLE)
    message(STATUS "IWYU: llvm-config not found trying usual prefix for finding LLVM/Clang modules.")
    set(IWYU_LLVM_CMAKE_DIR "${CMAKE_INSTALL_PREFIX}/lib/cmake/llvm" CACHE FILEPATH "Path to lib/cmake/llvm directory")
    set(IWYU_CLANG_CMAKE_DIR "${CMAKE_INSTALL_PREFIX}/lib/cmake/clang" CACHE FILEPATH "Path to lib/cmake/llvm directory")
  endif(LLVM_CONFIG_EXECUTABLE)

  if( NOT EXISTS ${IWYU_LLVM_CMAKE_DIR} OR NOT EXISTS ${IWYU_CLANG_CMAKE_DIR} )
    message(FATAL_ERROR
      "IWYU: Don't know how to find LLVM or Clang CMake modules. "
      "Use -DIWYU_LLVM_CMAKE_DIR=/abc and -DIWYU_CLANG_CMAKE_DIR=/xyz")
  endif()

  # Pull in LLVM and Clang CMake modules and add their build options
  list(APPEND CMAKE_MODULE_PATH ${IWYU_LLVM_CMAKE_DIR})
  list(APPEND CMAKE_MODULE_PATH ${IWYU_CLANG_CMAKE_DIR})
  find_package(LLVM REQUIRED CONFIG)
  find_package(Clang REQUIRED CONFIG)
  include(AddLLVM)
  include(HandleLLVMOptions)
else()
  # This the case when building IWYU in the LLVM/Clang tree.
  set(IWYU_IS_BUILT_STANDALONE FALSE)
  message(STATUS "IWYU: is built inside the LLVM/Clang tree.")
endif()

add_definitions(${LLVM_DEFINITIONS})

include_directories(
  ${LLVM_INCLUDE_DIRS}
  ${CLANG_INCLUDE_DIRS}
  )

# Pick up Git revision so we can report it in version information.
find_package(Git)
if( GIT_FOUND AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/.git" )
  execute_process(COMMAND ${GIT_EXECUTABLE} rev-parse --short HEAD
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    OUTPUT_VARIABLE IWYU_GIT_REV
    OUTPUT_STRIP_TRAILING_WHITESPACE)
else()
  message(STATUS "IWYU: Warning: IWYU Git version info not found, DO NOT release "
                 "from this build tree!")
endif()
add_definitions(-DIWYU_GIT_REV="${IWYU_GIT_REV}")

add_llvm_executable(include-what-you-use
  iwyu.cc
  iwyu_ast_util.cc
  iwyu_cache.cc
  iwyu_driver.cc
  iwyu_getopt.cc
  iwyu_globals.cc
  iwyu_include_picker.cc
  iwyu_lexer_utils.cc
  iwyu_location_util.cc
  iwyu_output.cc
  iwyu_path_util.cc
  iwyu_preprocessor.cc
  iwyu_verrs.cc
  )

# Check whether if LLVM shared lib is available from the LLVM [imported] targets
if (TARGET LLVM OR (NOT IWYU_IS_BUILT_STANDALONE AND LLVM_BUILD_LLVM_DYLIB))
  set(LLVM_LIBS LLVM)
  message(STATUS "IWYU: LLVM shared lib target available, using it.")
else()
  set(LLVM_LIBS LLVMSupport LLVMX86AsmParser LLVMX86Desc LLVMX86Info)
  message(STATUS "IWYU: LLVM shared lib target not found, using individual components targets: LLVM_LIBS=${LLVM_LIBS}")
endif()

target_link_libraries(include-what-you-use
  ${LLVM_LIBS}
  clangBasic
  clangLex
  clangAST
  clangSema
  clangFrontend
  clangDriver
  )

if( WIN32 )
  # For PathMatchSpecA to emulate fnmatch.
  target_link_libraries(include-what-you-use
    shlwapi
    )
endif()

install(TARGETS include-what-you-use RUNTIME DESTINATION bin)
install(PROGRAMS fix_includes.py iwyu_tool.py DESTINATION bin)

# Install mapping files
file(GLOB MAPPING_FILES *.imp)
install(FILES ${MAPPING_FILES} DESTINATION share/include-what-you-use)

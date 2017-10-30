cmake_minimum_required(VERSION 3.4.3)

if( POLICY CMP0048 )
  # Silence CMP0048 warning about missing project VERSION.
  cmake_policy(SET CMP0048 NEW)
endif()

project(include-what-you-use)

# TODO: Make these configurable
# set(IWYU_LLVM_CMAKE_DIR /home/kgrasman/code/llvm-trunk/build/lib/cmake/llvm)
# set(IWYU_CLANG_CMAKE_DIR /home/kgrasman/code/llvm-trunk/build/lib/cmake/clang)

if( NOT DEFINED IWYU_LLVM_CMAKE_DIR OR NOT DEFINED IWYU_CLANG_CMAKE_DIR )
  message(FATAL_ERROR
    "Don't know how to find LLVM or Clang CMake modules. "
    "Use -DIWYU_LLVM_CMAKE_DIR=/abc and -DIWYU_CLANG_CMAKE_DIR=/xyz")
endif()

# Pull in LLVM and Clang CMake modules and add their build options
set(LLVM_DIR ${IWYU_LLVM_CMAKE_DIR})
set(Clang_DIR ${IWYU_CLANG_CMAKE_DIR})
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${LLVM_DIR}")
find_package(LLVM REQUIRED CONFIG)
find_package(Clang REQUIRED CONFIG)
include(AddLLVM)
include(HandleLLVMOptions)

add_definitions(${LLVM_DEFINITIONS})

include_directories(
  ${LLVM_INCLUDE_DIRS}
  ${CLANG_INCLUDE_DIRS}
  )

# Pick up Git revision so we can report it in version information.
include(FindGit)
if( GIT_FOUND AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/.git" )
  execute_process(COMMAND ${GIT_EXECUTABLE} rev-parse --short HEAD
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    OUTPUT_VARIABLE IWYU_GIT_REV
    OUTPUT_STRIP_TRAILING_WHITESPACE)
else()
  message(STATUS "Warning: IWYU Git version info not found, DO NOT release "
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

target_link_libraries(include-what-you-use
  LLVMSupport
  LLVMX86AsmParser
  LLVMX86Desc
  LLVMX86Info
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

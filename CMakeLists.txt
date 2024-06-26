# ———————————————————————————
# standard options
# ———————————————————————————
cmake_minimum_required(VERSION 3.22)
# set(MY_FLAGS "-Wall -Wpedantic -Wshadow")
# set(MY_FLAGS_DEBUG "-fsanitize=address,undefined -O2")
# set(CMAKE_C_FLAGS_INIT ${MY_FLAGS})
# set(CMAKE_CXX_FLAGS_INIT ${MY_FLAGS})
# set(CMAKE_C_FLAGS_DEBUG_INIT ${MY_FLAGS_DEBUG})
# set(CMAKE_CXX_FLAGS_DEBUG_INIT ${MY_FLAGS_DEBUG})# Change CXX to C, if you're making a C program
project(cpptest DESCRIPTION "HypermodernCPP" LANGUAGES CXX) 
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

set(PROJECT_VERSION_MAJOR 1)
set(PROJECT_VERSION_MINOR 0)
set(PROJECT_VERSION_PATCH 0)
set(PROJECT_VERSION ${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}.${PROJECT_VERSION_PATCH})
#set(CMAKE_CXX_CLANG_TIDY clang-tidy -checks=-*,readability-*)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
option(ONLY_COVERAGE "Build only tests necessary for coverage" FALSE)
option(LIBCPP "Build with libc++" FALSE)
option(ENABLE_COVERAGE "Enable coverage reporting for gcc/clang" FALSE)
option(ENABLE_ASAN "Enable address sanitizer" FALSE)
option(BUILD_SHARED_LIBS "Enable compilation of shared libraries" FALSE)
option(ENABLE_TESTING "Enable the building of the test" TRUE)
option(ENABLE_CLANG_TIDY "Enable testing with clang-tidy" FALSE)
option(ENABLE_CPPCHECK "Enable testing with cppcheck" FALSE)
option(SIMPLE_BUILD "Build the project as minimally as possible" FALSE)
option(BUILD_DOC "Build the project's documentation" ON)
option(FORCE_COLORED_OUTPUT "Always produce ANSI-colored output (GNU/Clang only)." TRUE)
option(DEBUG_LOGGING "Enabling debug logging" FALSE)
OPTION(OPTIMIZE "Build the project with compiler optimization" FALSE)
OPTION(DEBUG  "Build the project in debugging mode"  FALSE)
OPTION(GPROF  "Build the project for gprof (requires DEBUG)"  FALSE)


set(CPACK_PROJECT_NAME ${PROJECT_NAME})
set(CPACK_PROJECT_VERSION ${PROJECT_VERSION})
set(CPACK_RESOURCE_FILE_README ${CMAKE_CURRENT_SOURCE_DIR}/README.md)
set(CPACK_SOURCE_IGNORE_FILES
    /.git
    /.vscode
    /.*build.*
    /\\\\.gitignore
    /\\\\.DS_Store
)
include(CPack)

if (MSVC)
    set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
endif()

set(VCPKG_TARGET_ARCHITECTURE x64)
set(VCPKG_CRT_LINKAGE dynamic)
set(VCPKG_LIBRARY_LINKAGE dynamic)

# Debug option that also enables asserts and profiling
if (DEBUG OR CMAKE_BUILD_TYPE STREQUAL "Debug")
  add_compile_definitions(
    APP_DEBUG APP_ENABLE_ASSERTS APP_PROFILE)
endif ()

# ———————————————————————————
# complier options
# ———————————————————————————
add_library(project_warnings INTERFACE)
add_library(project_options INTERFACE)

if(ONLY_COVERAGE OR ENABLE_COVERAGE)
    target_compile_options(project_options INTERFACE --coverage -O0 -g)
    target_link_libraries(project_options INTERFACE --coverage)
endif()

if(ENABLE_ASAN)
    target_compile_options(project_options INTERFACE -fsanitize=address)
    target_link_libraries(project_options INTERFACE -fsanitize=address)
endif()

IF(OPTIMIZE)
	SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3 -DNDEBUG")
ENDIF(OPTIMIZE)

IF(DEBUG)
	SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -DDEBUG -DPERF")
ENDIF(DEBUG)

IF(GPROF)
	SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pg -DGPROF")
	set(GPROFFLAG "-pg")
ENDIF(GPROF)

function(set_project_warnings project_name)
  # I always treat warnings as errors, making my life easier
  # in the long run.
  option(
    WARNINGS_AS_ERRORS
    "Treat compiler warnings as errors" TRUE)
  message(STATUS "Treat compiler warnings as errors")

  set(MSVC_WARNINGS
    # Baseline reasonable warnings
    /W4
    # Whatever other MSVC compiler warnings you want.
    # ...
    )

  set(CLANG_WARNINGS
    # Reasonable and standard
    -Wall
    -Wextra
    -Wshadow
    -Wnon-virtual-dtor
    -Wold-style-cast
    -Wcast-align
    -Wunused
    -Woverloaded-virtual
    -Wpedantic
    -Wconversion
    -Wsign-conversion
    -Wnull-dereference
    -Wdouble-promotion
    -Wformat=2
    )

  # Stop the build if there are any warnings,
  if (WARNINGS_AS_ERRORS)
    set(CLANG_WARNINGS ${CLANG_WARNINGS} -Werror)
    set(MSVC_WARNINGS ${MSVC_WARNINGS} /WX)
  endif ()

  set(GCC_WARNINGS
    ${CLANG_WARNINGS}
    -Wmisleading-indentation
    -Wduplicated-cond
    -Wlogical-op
    -Wuseless-cast
    )

  # Enable the right set of warnings depending on
  # the used compiler.
  if (MSVC)
    set(PROJECT_WARNINGS ${MSVC_WARNINGS})
  elseif (CMAKE_CXX_COMPILER_ID MATCHES ".*Clang")
    set(PROJECT_WARNINGS ${CLANG_WARNINGS})
  elseif (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    set(PROJECT_WARNINGS ${GCC_WARNINGS})
    target_link_libraries(project_options INTERFACE stdc++fs)	
  else ()
    message(
      AUTHOR_WARNING
      "No compiler warnings set for '${CMAKE_CXX_COMPILER_ID}' compiler.")
  endif ()

  # This will "link" the warnings to the defined
  # project name, in my case "project_warnings".
  target_compile_options(
    ${project_name} INTERFACE ${PROJECT_WARNINGS})
endfunction()

set_project_warnings(project_warnings)

if (${FORCE_COLORED_OUTPUT})
    if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
        target_compile_options (project_options INTERFACE -fdiagnostics-color=always)
    elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
        target_compile_options (project_options INTERFACE -fcolor-diagnostics)
    endif ()
endif ()

find_program(CCACHE ccache)

if(CCACHE)
    message("using ccache")
    set(CMAKE_CXX_COMPILER_LAUNCHER "${CCACHE}")
else()
    message("ccache not available")
endif()

include(CheckIPOSupported)

# Enable Interprocedural Optimisation if available
check_ipo_supported(RESULT result OUTPUT output)
if(result)
    message("using interprocedural optimisation")
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
else()
    message(SEND_ERROR "IPO not available: ${output}")
endif()

# Include clang-tidy if available
if(ENABLE_CLANG_TIDY)
    find_program(CLANGTIDY clang-tidy)
    if(CLANGTIDY)
    set(CMAKE_CXX_CLANG_TIDY ${CLANGTIDY})
    else()
    message(SEND_ERROR "clang-tidy requested but executable not found")
    endif()
endif()

# Include cppcheck if available
if(ENABLE_CPPCHECK)
    find_program(CPPCHECK cppcheck)
    if(CPPCHECK)
    set(CMAKE_CXX_CPPCHECK
        ${CPPCHECK}
        --suppress=syntaxError
        --enable=all
        --inconclusive)
    else()
    message(SEND_ERROR "cppcheck requested but executable not found")
    endif()
endif()
# ———————————————————————————
# documentation
# ———————————————————————————

if(BUILD_DOC)
    find_package(Doxygen)
    if (DOXYGEN_FOUND)
                SET(DOXYGEN_IN ${CMAKE_CURRENT_SOURCE_DIR}/doc/Doxyfile.in)
                SET(DOXYGEN_OUT ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile)

                configure_file(${DOXYGEN_IN} ${DOXYGEN_OUT} @ONLY)

                add_custom_target(doc ALL
                        COMMAND ${DOXYGEN_EXECUTABLE} ${DOXYGEN_OUT}
                        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
                        COMMENT "Buidling Doxygen documentation"
                        VERBATIM )
    else (DOXYGEN_FOUND)
                message("No doxygen binary found on the system.")
                SET(${BUILD_DOC} OFF)
    endif ()
endif()

# ———————————————————————————
# testing
# ———————————————————————————
if(ENABLE_TESTING)
    enable_testing()
endif()

add_subdirectory(tests)
#find_package(GTest REQUIRED)

# Include Google Test module
#include(GoogleTest)

# —————————————————————————
# create target for library
# —————————————————————————
message("###############################################################")
message("\t \t                  Summary")
message("###############################################################")
message("Build type:       \t ${CMAKE_BUILD_TYPE}")
message("Install prefix:   \t ${CMAKE_INSTALL_PREFIX}")
message("Testing enabled:  \t ${ENABLE_TESTING}")
message("Clang-tidy:       \t ${ENABLE_CLANG_TIDY}")
message("Cppcheck:         \t ${ENABLE_CPPCHECK}")
message("Compiler:         \t ${CMAKE_CXX_COMPILER_ID}")
message("Sanizizers:       \t ${ENABLE_ASAN}")
message("Shared libs:      \t ${BUILD_SHARED_LIBS}")
message("Build libcpp:     \t ${LIBCPP}")
message("CCache executable:\t ${CCACHE}")
message("Building doc:     \t ${BUILD_DOC}")
message("---------------------------------------------------------------")
message("Version:          \t ${PROJECT_VERSION}")

message("C complier:       \t ${CMAKE_C_COMPILER}")
message("C++ compiler:     \t ${CMAKE_CXX_COMPILER}")
message("Compiler id:      \t ${CMAKE_CXX_COMPILER_ID}")
message("Compiler version: \t ${CMAKE_CXX_COMPILER_VERSION}")
message("###############################################################")

# Change *.cpp to *.c, if you're making a C program

# file(GLOB PROJECT_SOURCE_FILES CONFIGURE_DEPENDS src/*.cpp)

# include_directories(include/)
add_subdirectory(src/customlib)
# —————————————————————————————
# create target for main binary
# —————————————————————————————
add_executable(main src/main.cpp)

# install(FILES Foo.hpp DESTINATION include)
# install(TARGETS foo DESTINATION lib)
# install(FILES FooConfig.cmake DESTINATION lib/cmake/Foo)
# -- Install configuration: ""
# -- Installing: /usr/local/include/Foo.hpp
# -- Installing: /usr/local/lib/libfoo.a
# -- Installing: /usr/local/lib/cmake/Foo/FooConfig.cmake
target_link_libraries(main PRIVATE customlib)

# ———————————————————————————
# find 3rd party dependencies using vcpkg
# ———————————————————————————
# set(CMAKE_MODULE_PATH "/usr/local/lib/SomeLib/cmake/;${CMAKE_MODULE_PATH}")
# find_package(SomeLib REQUIRED)

find_package(fmt CONFIG REQUIRED)
target_link_libraries(main PRIVATE fmt::fmt)

# Or use the header-only version
# find_package(fmt CONFIG REQUIRED)
# target_link_libraries(main PRIVATE fmt::fmt-header-only)
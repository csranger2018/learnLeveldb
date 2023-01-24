cmake_minimum_required(VERSION 3.22)
# Keep the version below in sync with the one in db.h
project(leveldb VERSION 1.23.0 LANGUAGES C CXX)

# C standard can be overridden when this is used as a sub-project.
if(NOT CMAKE_C_STANDARD)
    # This project can use C11, but will gracefully decay down to C89.
    set(CMAKE_C_STANDARD 11)
    set(CMAKE_C_STANDARD_REQUIRED OFF)
    set(CMAKE_C_EXTENSIONS OFF)
endif(NOT CMAKE_C_STANDARD)

# C++ standard can be overridden when this is used as a sub-project.
# 可以-DCMAKE_CXX_STANDARD=xx在命令行上提供（对于 11、14、17）
# 检查 CMAKE_CXX_STANDARD 变量是否设置
if(NOT CMAKE_CXX_STANDARD)
    # This project requires C++11.
    set(CMAKE_CXX_STANDARD 11)
    set(CMAKE_CXX_STANDARD_REQUIRED ON)
    set(CMAKE_CXX_EXTENSIONS OFF)
endif(NOT CMAKE_CXX_STANDARD)

set(LEVELDB_PLATFORM_NAME LEVELDB_PLATFORM_POSIX)

# option(<variable> "<help_text>" [value])
# If no initial <value> is provided, boolean OFF is the default value.
option(LEVELDB_BUILD_TESTS "Build LevelDB's unit tests" ON)
option(LEVELDB_BUILD_BENCHMARKS "Build LevelDB's benchmarks" ON)
option(LEVELDB_INSTALL "Install LevelDB's header and library" ON)

# CheckIncludeFile: Provides a macro to check if a header file can be included in C.
include(CheckIncludeFile)
check_include_file("unistd.h" HAVE_UNISTD_H)

# CheckLibraryExists: Check if the function exists.
# CHECK_LIBRARY_EXISTS(LIBRARY FUNCTION LOCATION VARIABLE)
# LIBRARY  - the name of the library you are looking for
# FUNCTION - the name of the function
# LOCATION - location where the library should be found
# VARIABLE - variable to store the result
#            Will be created as an internal cache variable.
include(CheckLibraryExists)
check_library_exists(crc32c crc32c_value "" HAVE_CRC32C)
check_library_exists(snappy snappy_compress "" HAVE_SNAPPY)
check_library_exists(tcmalloc malloc "" HAVE_TCMALLOC)

include(CheckCXXSymbolExists)
# Using check_cxx_symbol_exists() instead of check_c_symbol_exists() because
# we're including the header from C++, and feature detection should use the same
# compiler language that the project will use later. Principles aside, some
# versions of do not expose fdatasync() in <unistd.h> in standard C mode
# (-std=c11), but do expose the function in standard C++ mode (-std=c++11).
check_cxx_symbol_exists(fdatasync "unistd.h" HAVE_FDATASYNC)
check_cxx_symbol_exists(F_FULLFSYNC "fcntl.h" HAVE_FULLFSYNC)
check_cxx_symbol_exists(O_CLOEXEC "fcntl.h" HAVE_O_CLOEXEC)

# CMAKE_<LANG>_FLAGS: Flags for all build types.
# message: Log a message.
# message([<mode>] "message text" ...)
# Enable strict prototype warnings for C code in clang and gcc.
if(NOT CMAKE_C_FLAGS MATCHES "-Wstrict-prototypes")
    message(STATUS "CMAKE_C_FLAGS = ${CMAKE_C_FLAGS}")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wstrict-prototypes")
    message(STATUS "CMAKE_C_FLAGS = ${CMAKE_C_FLAGS}")
endif(NOT CMAKE_C_FLAGS MATCHES "-Wstrict-prototypes")

# Disable C++ exceptions.
message(STATUS "CMAKE_CXX_FLAGS = ${CMAKE_CXX_FLAGS}")
string(REGEX REPLACE "-fexceptions" "" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-exceptions")
message(STATUS "CMAKE_CXX_FLAGS = ${CMAKE_CXX_FLAGS}")

# Disable RTTI.
string(REGEX REPLACE "-frtti" "" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-rtti")

# Test whether -Wthread-safety is available. See
# https://clang.llvm.org/docs/ThreadSafetyAnalysis.html
include(CheckCXXCompilerFlag)
check_cxx_compiler_flag(-Wthread-safety HAVE_CLANG_THREAD_SAFETY)

# Used by googletest.
check_cxx_compiler_flag(-Wno-missing-field-initializers
        LEVELDB_HAVE_NO_MISSING_FIELD_INITIALIZERS)


include(CheckCXXSourceCompiles)

# Test whether C++17 __has_include is available.
check_cxx_source_compiles("
#if defined(__has_include) &&  __has_include(<string>)
#include <string>
#endif
int main() { std::string str; return 0; }
" HAVE_CXX17_HAS_INCLUDE)

set(LEVELDB_PUBLIC_INCLUDE_DIR "include/leveldb")
set(LEVELDB_PORT_CONFIG_DIR "include/port")

configure_file(
        "port/port_config.h.in"
        "${PROJECT_BINARY_DIR}/${LEVELDB_PORT_CONFIG_DIR}/port_config.h"
)

include_directories(
        "${PROJECT_BINARY_DIR}/include"
        "."
)

if(BUILD_SHARED_LIBS)
    # Only export LEVELDB_EXPORT symbols from the shared library.
    add_compile_options(-fvisibility=hidden)
endif(BUILD_SHARED_LIBS)

# Must be included before CMAKE_INSTALL_INCLUDEDIR is used.
include(GNUInstallDirs)
message(STATUS "CMAKE_INSTALL_INCLUDEDIR = ${CMAKE_INSTALL_INCLUDEDIR}")
message(STATUS "CMAKE_INSTALL_FULL_INCLUDEDIR = ${CMAKE_INSTALL_FULL_INCLUDEDIR}")

add_library(leveldb "")
target_sources(leveldb
        PRIVATE
        "${PROJECT_BINARY_DIR}/${LEVELDB_PORT_CONFIG_DIR}/port_config.h"
        "port/port_stdcxx.h"
        "port/port.h"
        "port/thread_annotations.h"
        "util/coding.cc"
        "util/coding.h"
        "util/random.h"
        "util/mutexlock.h"

        # Only CMake 3.3+ supports PUBLIC sources in targets exported by "install".
        $<$<VERSION_GREATER:CMAKE_VERSION,3.2>:PUBLIC>
        "${LEVELDB_PUBLIC_INCLUDE_DIR}/comparator.h"
        "${LEVELDB_PUBLIC_INCLUDE_DIR}/iterator.h"
        "${LEVELDB_PUBLIC_INCLUDE_DIR}/slice.h"
        "${LEVELDB_PUBLIC_INCLUDE_DIR}/status.h"
        "${LEVELDB_PUBLIC_INCLUDE_DIR}/export.h"
        "${LEVELDB_PUBLIC_INCLUDE_DIR}/env.h"
        )

# MemEnv is not part of the interface and could be pulled to a separate library.
target_sources(leveldb
        PRIVATE
        "helpers/memenv/memenv.cc"
        "helpers/memenv/memenv.h"
        )

target_include_directories(leveldb
        PUBLIC
        $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
        )

# debug generator
file(GENERATE OUTPUT "./generator.txt" CONTENT
        "$<$<CONFIG:Debug>:-g -o0> \n$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
$<CONFIG>")


set_target_properties(leveldb
        PROPERTIES VERSION ${PROJECT_VERSION} SOVERSION ${PROJECT_VERSION_MAJOR})

# target_compile_definitions: Add compile definitions to a target.
# -DLEVELDB_COMPILE_LIBRARY -DLEVELDB_PLATFORM_POSIX=1
target_compile_definitions(leveldb
        PRIVATE
        # Used by include/export.h when building shared libraries.
        LEVELDB_COMPILE_LIBRARY
        # Used by port/port.h.
        ${LEVELDB_PLATFORM_NAME}=1
        )
if (NOT HAVE_CXX17_HAS_INCLUDE)
    target_compile_definitions(leveldb
            PRIVATE
            LEVELDB_HAS_PORT_CONFIG_H=1
            )
endif(NOT HAVE_CXX17_HAS_INCLUDE)

if(BUILD_SHARED_LIBS)
    target_compile_definitions(leveldb
            PUBLIC
            # Used by include/export.h.
            LEVELDB_SHARED_LIBRARY
            )
endif(BUILD_SHARED_LIBS)

if(HAVE_CLANG_THREAD_SAFETY)
    target_compile_options(leveldb
            PUBLIC
            -Werror -Wthread-safety)
endif(HAVE_CLANG_THREAD_SAFETY)

if(HAVE_CRC32C)
    target_link_libraries(leveldb crc32c)
endif(HAVE_CRC32C)
if(HAVE_SNAPPY)
    target_link_libraries(leveldb snappy)
endif(HAVE_SNAPPY)
if(HAVE_TCMALLOC)
    target_link_libraries(leveldb tcmalloc)
endif(HAVE_TCMALLOC)

# Needed by port_stdcxx.h
find_package(Threads REQUIRED)
target_link_libraries(leveldb Threads::Threads)


if(LEVELDB_BUILD_TESTS)
    enable_testing()

    # Prevent overriding the parent project's compiler/linker settings on Windows.
    set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
    set(install_gtest OFF)
    set(install_gmock OFF)
    set(build_gmock ON)

    # This project is tested using GoogleTest.
    add_subdirectory("third_party/googletest")

    # GoogleTest triggers a missing field initializers warning.
    if(LEVELDB_HAVE_NO_MISSING_FIELD_INITIALIZERS)
        set_property(TARGET gtest
                APPEND PROPERTY COMPILE_OPTIONS -Wno-missing-field-initializers)
        set_property(TARGET gmock
                APPEND PROPERTY COMPILE_OPTIONS -Wno-missing-field-initializers)
    endif(LEVELDB_HAVE_NO_MISSING_FIELD_INITIALIZERS)

    function(leveldb_test test_file)
        get_filename_component(test_target_name "${test_file}" NAME_WE)

        add_executable("${test_target_name}" "")
        target_sources("${test_target_name}"
                PRIVATE
                "${PROJECT_BINARY_DIR}/${LEVELDB_PORT_CONFIG_DIR}/port_config.h"

                "${test_file}"
                )
        target_link_libraries("${test_target_name}" leveldb gmock gtest)
        target_compile_definitions("${test_target_name}"
                PRIVATE
                ${LEVELDB_PLATFORM_NAME}=1
                )
        if (NOT HAVE_CXX17_HAS_INCLUDE)
            target_compile_definitions("${test_target_name}"
                    PRIVATE
                    LEVELDB_HAS_PORT_CONFIG_H=1
                    )
        endif(NOT HAVE_CXX17_HAS_INCLUDE)

        add_test(NAME "${test_target_name}" COMMAND "${test_target_name}")
    endfunction(leveldb_test)

    if(NOT BUILD_SHARED_LIBS)
        leveldb_test("util/coding_test.cc")
    endif(NOT BUILD_SHARED_LIBS)
endif(LEVELDB_BUILD_TESTS)


if(LEVELDB_INSTALL)
    message(STATUS "CMAKE_INSTALL_BINDIR = ${CMAKE_INSTALL_BINDIR} CMAKE_INSTALL_LIBDIR = ${CMAKE_INSTALL_LIBDIR}")
    install(TARGETS leveldb
            EXPORT leveldbTargets
            RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
            LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
            ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
            )
    install(
            FILES
            "${LEVELDB_PUBLIC_INCLUDE_DIR}/comparator.h"
            "${LEVELDB_PUBLIC_INCLUDE_DIR}/iterator.h"
            "${LEVELDB_PUBLIC_INCLUDE_DIR}/slice.h"
            "${LEVELDB_PUBLIC_INCLUDE_DIR}/status.h"
            "${LEVELDB_PUBLIC_INCLUDE_DIR}/export.h"
            "${LEVELDB_PUBLIC_INCLUDE_DIR}/env.h"
            DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/leveldb"
    )

    include(CMakePackageConfigHelpers)
    configure_package_config_file(
            "cmake/${PROJECT_NAME}Config.cmake.in"
            "${PROJECT_BINARY_DIR}/cmake/${PROJECT_NAME}Config.cmake"
            INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"
    )
    write_basic_package_version_file(
            "${PROJECT_BINARY_DIR}/cmake/${PROJECT_NAME}ConfigVersion.cmake"
            COMPATIBILITY SameMajorVersion
    )
    install(
            EXPORT leveldbTargets
            NAMESPACE leveldb::
            DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"
    )
    install(
            FILES
            "${PROJECT_BINARY_DIR}/cmake/${PROJECT_NAME}Config.cmake"
            "${PROJECT_BINARY_DIR}/cmake/${PROJECT_NAME}ConfigVersion.cmake"
            DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"
    )
endif(LEVELDB_INSTALL)

cmake_minimum_required(VERSION 3.0)
project(Pink VERSION 2.0 LANGUAGES C CXX)

message(STATUS "${PROJECT_NAME} version ${PROJECT_VERSION}")

# Set release as default build type
if (NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release")
endif()

message(STATUS "CMake build type: ${CMAKE_BUILD_TYPE}")

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -std=c++14 -fPIC")

if(${CMAKE_CXX_COMPILER} MATCHES "clang")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-missing-braces -Wmissing-field-initializers")
endif()

find_package(OpenMP REQUIRED)
if(OPENMP_FOUND)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
endif()

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)

# Download conan-cmake
find_program(CONAN conan)
if(CONAN)
    if(NOT EXISTS "${CMAKE_BINARY_DIR}/conan.cmake")
       message(STATUS "Downloading conan.cmake from https://github.com/conan-io/cmake-conan")
       file(DOWNLOAD "https://raw.githubusercontent.com/conan-io/cmake-conan/v0.13/conan.cmake"
                     "${CMAKE_BINARY_DIR}/conan.cmake")
    endif()
    
    # Conan setup
    include(${CMAKE_BINARY_DIR}/conan.cmake)
    conan_cmake_run(CONANFILE conanfile.py
                    BASIC_SETUP CMAKE_TARGETS
                    SETTINGS compiler.libcxx=libstdc++11
                    BUILD missing)
    
    include(${CMAKE_BINARY_DIR}/conanbuildinfo.cmake)
    conan_basic_setup()
endif()

# Get Git revision
if(EXISTS ${PROJECT_SOURCE_DIR}/.git)
    find_package(Git)
    if(GIT_FOUND)
        execute_process(
            COMMAND ${GIT_EXECUTABLE} rev-parse --short HEAD
            WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}"
            OUTPUT_VARIABLE "GIT_REVISION"
            ERROR_QUIET
            OUTPUT_STRIP_TRAILING_WHITESPACE)
        message(STATUS "Git revision: ${GIT_REVISION}")
    else()
        set(GIT_REVISION 0)
    endif()
endif()

# Forward Pink version and Git revision into code
configure_file (
    "${PROJECT_SOURCE_DIR}/src/UtilitiesLib/Version.h.in"
    "${PROJECT_SOURCE_DIR}/src/UtilitiesLib/Version.h"
)

# set RPATH for build and installed executable
set(CMAKE_SKIP_BUILD_RPATH FALSE)
set(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE) 
set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
list(FIND CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/lib" isSystemDir)
if("${isSystemDir}" STREQUAL "-1")
    set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
endif()

find_package(CUDA)
if(CUDA_FOUND)
    set(PINK_USE_CUDA true)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DPINK_USE_CUDA")
    # Set flags for CUDA compiler
    list(APPEND CUDA_NVCC_FLAGS "
    --expt-extended-lambda
    --expt-relaxed-constexpr
    -arch=sm_61
     -gencode=arch=compute_50,code=sm_50
     -gencode=arch=compute_52,code=sm_52
     -gencode=arch=compute_60,code=sm_60
     -gencode=arch=compute_61,code=sm_61
     -gencode=arch=compute_70,code=sm_70
     -gencode=arch=compute_70,code=compute_70
    ")
endif()

set(PYBIND11_PYTHON_VERSION 3)
find_package(pybind11)
if(pybind11_FOUND)
    add_subdirectory(src/PythonBinding)
endif()

add_subdirectory(src)

find_package(GTest)
if(GTEST_FOUND)
    ENABLE_TESTING()
    add_subdirectory(test)
endif()

find_package(Doxygen)
if(DOXYGEN_FOUND)
    configure_file(${PROJECT_SOURCE_DIR}/doxygen/Doxyfile
                   ${PROJECT_BINARY_DIR}/doxygen/Doxyfile)
    add_custom_target(
        doc
        ${DOXYGEN_EXECUTABLE} ${PROJECT_BINARY_DIR}/doxygen/Doxyfile
            WORKING_DIRECTORY ${PROJECT_BINARY_DIR}/doxygen
            COMMENT "Generating API documentation with Doxygen" VERBATIM
    )
endif()

install(DIRECTORY scripts/
    DESTINATION scripts
    FILES_MATCHING PATTERN "*.py"
    PERMISSIONS OWNER_EXECUTE OWNER_WRITE OWNER_READ
                GROUP_EXECUTE GROUP_READ
                WORLD_EXECUTE WORLD_READ
)

if(UNIX)
    set(CPACK_CMAKE_GENERATOR "Unix Makefiles")
    set(CPACK_SOURCE_GENERATOR "TGZ;TBZ2")
    set(CPACK_GENERATOR "TGZ;TBZ2;DEB;RPM")
    set(CPACK_PACKAGE_MAINTAINER "Bernd Doser")
    set(CPACK_PACKAGE_VENDOR "HITS gGmbH - Astroinformatics")
    set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "Parallelized rotation and flipping INvariant Kohonen maps")
    set(CPACK_PACKAGE_DESCRIPTION_FILE "${CMAKE_CURRENT_SOURCE_DIR}/README.md")
    set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE")
    set(CPACK_PACKAGE_VERSION "${PROJECT_VERSION}")
    set(CPACK_PACKAGE_CONTACT "Bernd Doser <bernd.doser@h-its.org>")
    include(CPack)
endif()

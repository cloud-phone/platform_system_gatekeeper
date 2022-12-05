cmake_policy(SET CMP0048 NEW)
cmake_policy(SET CMP0079 NEW)
get_directory_property(hasParent PARENT_DIRECTORY)
if(NOT hasParent)
    project(gatekeeper)
endif()
cmake_minimum_required(VERSION 3.18)


set(VALID_DEPENDENCY_RESOLUTION AOSP SYSTEM DOWNLOAD)
set(DEPENDENCY_RESOLUTION "AOSP"  CACHE STRING "\
How to resolve the dependencies. Currently there are 3 options: AOSP, SYSTEM and DOWNLOAD. AOSP \
will assume the repo is checked out in an Android tree, and find the dependencies through \
relative paths. SYSTEM will use the cmake find_package to identify the package. DOWNLOAD will use \
CMake FetchContent to download the dependencies from the AOSP tree.\
")
set_property(CACHE DEPENDENCY_RESOLUTION PROPERTY STRINGS ${VALID_DEPENDENCY_RESOLUTION})
if(NOT DEPENDENCY_RESOLUTION IN_LIST VALID_DEPENDENCY_RESOLUTION)
    message(FATAL_ERROR "DEPENDENCY_RESOLUTION must be one of ${VALID_DEPENDENCY_RESOLUTION}.")
endif()

# set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})
set(CMAKE_INSTALL_PREFIX ${CMAKE_BINARY_DIR}/distribution)
if (WIN32)
    SET(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /Zi")
else()
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g3 -O3")
endif()
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_C_STANDARD 11)

include(ExternalProject)
include(FetchContent)
set(FETCHCONTENT_QUIET FALSE)
set(PROJECT_TO_FETCH)
if(NOT TARGET libhardware)
    if(DEPENDENCY_RESOLUTION STREQUAL "AOSP")
    set(FMT_PATH ${PROJECT_SOURCE_DIR}/../../../hardware/libhardware)
    if(NOT EXISTS ${FMT_PATH})
        message(FATAL_ERROR "HARDWARE_PATH is not found.")
    endif()
    elseif(DEPENDENCY_RESOLUTION STREQUAL "DOWNLOAD")
    FetchContent_Declare(
        libhardware
        GIT_REPOSITORY https://android.googlesource.com/platform/hardware/libhardware
        GIT_REMOTE_NAME aosp
        GIT_TAG aosp/master
        GIT_REMOTE_UPDATE_STRATEGY CHECKOUT
        GIT_PROGRESS TRUE
    )
    list(APPEND PROJECT_TO_FETCH libhardware)
    add_library(hardware_headers INTERFACE)
    target_include_directories(hardware_headers INTERFACE ${FETCHCONTENT_BASE_DIR}/libhardware-src/include)
    endif()
endif()

if(PROJECT_TO_FETCH)
    FetchContent_MakeAvailable(${PROJECT_TO_FETCH})
endif()

add_library(
    gatekeeper
    SHARED
    gatekeeper_messages.cpp
    gatekeeper.cpp
)
target_include_directories(
    gatekeeper
    PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)
target_link_libraries(
    gatekeeper
    PRIVATE
    hardware_headers
)
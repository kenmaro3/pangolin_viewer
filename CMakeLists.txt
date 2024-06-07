cmake_minimum_required(VERSION 3.1)
project(pangolin_viewer LANGUAGES CXX C)

# OpenMP
if(APPLE)
    # On macOS, use Homebrew's OpenMP
    execute_process(COMMAND brew --prefix libomp
                    OUTPUT_VARIABLE OpenMP_HOME
                    OUTPUT_STRIP_TRAILING_WHITESPACE)
    if(EXISTS "${OpenMP_HOME}")
        set(OpenMP_C_FLAGS "-Xpreprocessor -fopenmp -I${OpenMP_HOME}/include")
        set(OpenMP_CXX_FLAGS "-Xpreprocessor -fopenmp -I${OpenMP_HOME}/include")
        set(OpenMP_C_LIB_NAMES "omp")
        set(OpenMP_CXX_LIB_NAMES "omp")
        set(OpenMP_omp_LIBRARY "${OpenMP_HOME}/lib/libomp.dylib")
        link_directories("${OpenMP_HOME}/lib")
        
        # Add these lines for CMake to recognize OpenMP flags
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
        set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${OpenMP_C_FLAGS}")

        # Add these for CMake to recognize OpenMP libraries
        set(CMAKE_REQUIRED_LIBRARIES "${OpenMP_omp_LIBRARY}")
    else()
        message(FATAL_ERROR "OpenMP not found!")
    endif()
else()
    find_package(OpenMP REQUIRED)
    if(NOT OpenMP_FOUND)
        message(FATAL_ERROR "OpenMP not found!")
    endif()
endif()

# ----- Find dependencies -----

# Manually specify GLEW paths
set(GLEW_INCLUDE_DIRS "/opt/homebrew/include")
set(GLEW_LIBRARIES "/opt/homebrew/lib/libGLEW.dylib")

find_package(stella_vslam REQUIRED)
find_package(Pangolin REQUIRED)

# GLEW
include_directories(${GLEW_INCLUDE_DIRS})
link_directories(${GLEW_LIBRARIES})

# ----- Set build type -----

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE "Release")
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release")
endif()
message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")

option(BUILD_SHARED_LIBS "Build stella_vslam as a shared library" ON)

# ----- Configure PangolinViewer library -----

add_library(pangolin_viewer
    ${CMAKE_CURRENT_SOURCE_DIR}/src/viewer.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/src/color_scheme.cc)

set_target_properties(pangolin_viewer PROPERTIES
    OUTPUT_NAME pangolin_viewer
    ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib
    LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
if(${Pangolin_VERSION} VERSION_GREATER 0.5)
    message(STATUS "Build pangolin_viewer with c++17")
    set_property(TARGET pangolin_viewer PROPERTY CXX_STANDARD 17)
endif()

target_include_directories(pangolin_viewer 
    PUBLIC
    "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include/>"
    "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/3rd/tinycolormap/include>"
    "$<INSTALL_INTERFACE:include/>"
    ${GLEW_INCLUDE_DIRS})

if(${Pangolin_VERSION} VERSION_GREATER 0.5)
    target_link_libraries(pangolin_viewer
        PUBLIC
        stella_vslam::stella_vslam
        opencv_highgui
        pango_display
        pango_opengl
        ${OpenMP_omp_LIBRARY}
        ${GLEW_LIBRARIES}
    )
else()
    target_link_libraries(pangolin_viewer
        PUBLIC
        stella_vslam::stella_vslam
        opencv_highgui
        pangolin
        ${OpenMP_omp_LIBRARY}
        ${GLEW_LIBRARIES}
    )
endif()

# ----- Install configuration -----

include(CMakePackageConfigHelpers)

set(PANGOLIN_VIEWER_GENERATED_DIR ${CMAKE_CURRENT_BINARY_DIR}/generated)
set(PANGOLIN_VIEWER_VERSION_CONFIG ${PANGOLIN_VIEWER_GENERATED_DIR}/${PROJECT_NAME}ConfigVersion.cmake)
set(PANGOLIN_VIEWER_PROJECT_CONFIG ${PANGOLIN_VIEWER_GENERATED_DIR}/${PROJECT_NAME}Config.cmake)
set(PANGOLIN_VIEWER_TARGETS_EXPORT_NAME ${PROJECT_NAME}Targets)
set(PANGOLIN_VIEWER_CONFIG_INSTALL_DIR lib/cmake/${PROJECT_NAME})
set(PANGOLIN_VIEWER_NAMESPACE "${PROJECT_NAME}::")
set(PANGOLIN_VIEWER_VERSION 0.0.1)

# Create a version config file
write_basic_package_version_file(${PANGOLIN_VIEWER_VERSION_CONFIG}
    VERSION ${PANGOLIN_VIEWER_VERSION}
    COMPATIBILITY SameMajorVersion)

configure_file(${PROJECT_SOURCE_DIR}/cmake/Config.cmake.in ${PANGOLIN_VIEWER_PROJECT_CONFIG} @ONLY)

# Install to the specified directory
install(FILES ${PANGOLIN_VIEWER_VERSION_CONFIG} ${PANGOLIN_VIEWER_PROJECT_CONFIG}
    DESTINATION ${PANGOLIN_VIEWER_CONFIG_INSTALL_DIR})
install(EXPORT ${PANGOLIN_VIEWER_TARGETS_EXPORT_NAME}
    NAMESPACE ${PANGOLIN_VIEWER_NAMESPACE}
    DESTINATION ${PANGOLIN_VIEWER_CONFIG_INSTALL_DIR})

# Set standard installation directories
set(RUNTIME_DESTINATION bin)
set(LIBRARY_DESTINATION lib)
set(ARCHIVE_DESTINATION lib)
set(INCLUDES_DESTINATION include)

set(PANGOLIN_VIEWER_INCLUDE_INSTALL_DIR ${INCLUDES_DESTINATION}/pangolin_viewer)

install(TARGETS pangolin_viewer
    EXPORT ${PANGOLIN_VIEWER_TARGETS_EXPORT_NAME}
    RUNTIME DESTINATION ${RUNTIME_DESTINATION}
    LIBRARY DESTINATION ${LIBRARY_DESTINATION}
    ARCHIVE DESTINATION ${ARCHIVE_DESTINATION}
    INCLUDES DESTINATION ${PANGOLIN_VIEWER_INCLUDE_INSTALL_DIR})

file(GLOB HEADERS "${CMAKE_CURRENT_SOURCE_DIR}/include/pangolin_viewer/*.h")
install(FILES ${HEADERS}
    DESTINATION ${PANGOLIN_VIEWER_INCLUDE_INSTALL_DIR})


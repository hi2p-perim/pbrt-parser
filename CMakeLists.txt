## ======================================================================== ##
## Copyright 2009-2019 Ingo Wald                                            ##
##                                                                          ##
## Licensed under the Apache License, Version 2.0 (the "License");          ##
## you may not use this file except in compliance with the License.         ##
## You may obtain a copy of the License at                                  ##
##                                                                          ##
##     http://www.apache.org/licenses/LICENSE-2.0                           ##
##                                                                          ##
## Unless required by applicable law or agreed to in writing, software      ##
## distributed under the License is distributed on an "AS IS" BASIS,        ##
## WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. ##
## See the License for the specific language governing permissions and      ##
## limitations under the License.                                           ##
## ======================================================================== ##

cmake_minimum_required(VERSION 3.10)

# Enable policy to manage VERSION in project() command
cmake_policy(SET CMP0048 NEW)
project(pbrtParser VERSION 2.2.0)

# Check if the directory is added via add_subdirectory
set(PBRTPARSER_MASTER_PROJECT OFF)
if (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
    set(PBRTPARSER_MASTER_PROJECT ON)
endif()

# Build options
option(PBRTPARSER_INSTALL "Enable install" ${PBRTPARSER_MASTER_PROJECT})

# For CMAKE_INSTALL_*
include(GNUInstallDirs)

# -----------------------------------------------------------------------------

# Main library

# Headers and sources
set(_HEADER_DIR "${CMAKE_CURRENT_SOURCE_DIR}/pbrtParser/include/pbrtParser")
set(_HEADER_FILES 
    "${_HEADER_DIR}/math.h"
    "${_HEADER_DIR}/Scene.h"
)
set(_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/pbrtParser/impl")
set(_SOURCE_FILES 
    "${_SOURCE_DIR}/syntactic/Lexer.h"
    "${_SOURCE_DIR}/syntactic/Lexer.cpp"
    "${_SOURCE_DIR}/syntactic/Parser.h"
    "${_SOURCE_DIR}/syntactic/Parser.cpp"
    "${_SOURCE_DIR}/syntactic/Scene.h"
    "${_SOURCE_DIR}/syntactic/Scene.cpp"
    "${_SOURCE_DIR}/semantic/SemanticParser.h"
    "${_SOURCE_DIR}/semantic/Geometry.cpp"
    "${_SOURCE_DIR}/semantic/Camera.cpp"
    "${_SOURCE_DIR}/semantic/Textures.cpp"
    "${_SOURCE_DIR}/semantic/Materials.cpp"
    "${_SOURCE_DIR}/semantic/Lights.cpp"
    "${_SOURCE_DIR}/semantic/Scene.cpp"
    "${_SOURCE_DIR}/semantic/BinaryFileFormat.cpp"
    "${_SOURCE_DIR}/semantic/importPBRT.cpp"
)
# Currently only build shared library
add_library(pbrtParser SHARED ${_HEADER_FILES} ${_SOURCE_FILES})
add_library(${PROJECT_NAME}::pbrtParser ALIAS pbrtParser)
target_include_directories(pbrtParser PUBLIC 
    "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/pbrtParser/include>"
    "$<INSTALL_INTERFACE:include>")
target_compile_definitions(pbrtParser PRIVATE PBRT_PARSER_DLL_INTERFACE)
target_compile_features(pbrtParser PRIVATE cxx_std_11)
set_target_properties(pbrtParser PROPERTIES
    ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib"
    LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin")
if (PBRTPARSER_INSTALL)
    install(
        TARGETS pbrtParser
        EXPORT ${PROJECT_NAME}Targets
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    )
endif()

# -----------------------------------------------------------------------------

# Example applications

set(_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/apps")

# '.pbrt' to '.obj' converter
add_executable(pbrt2obj "${_SOURCE_DIR}/pbrt2obj.cpp")
target_link_libraries(pbrt2obj PRIVATE pbrtParser)

# '.pbrt' to '.pbf' (pbrt binary format) converter
add_executable(pbrt2pbf "${_SOURCE_DIR}/pbrt2pbf.cpp")
target_link_libraries(pbrt2pbf PRIVATE pbrtParser)

# simple example of loading a pbrt or pbf file, finding all triangle
# meshes, and dumping all triangles in obj format. Note this is a
# intentionally simplisitic example that lacks support for many things
# (including, among other, vertex normals, texture coordinates,
# materials, textures, etc) - it ONLY dumps triangles, nothing
# else. Also be warned: since OBJ doesn't support instancing this can
# produce preeeeetty big output files for heavily instanced scenes
# (ie, do not try this on landscape, ecosys, or moana ...)
add_executable(pbrtInfo "${_SOURCE_DIR}/pbrtInfo.cpp")
target_link_libraries(pbrtInfo PRIVATE pbrtParser)

set_target_properties(pbrt2obj pbrt2pbf pbrtInfo PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin")
if (PBRTPARSER_INSTALL)     
    install(
        TARGETS pbrt2obj pbrt2pbf pbrtInfo
        EXPORT ${PROJECT_NAME}Targets
        LIBRARY DESTINATION ${CMAKE_INSTALL_BINDIR}
    )
endif()

# -----------------------------------------------------------------------------

# Install
if (PBRTPARSER_INSTALL)
    # Install include directory
    install(DIRECTORY "pbrtParser/include/pbrtParser" DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

    # Package configuration directory
    set(_PACKAGE_CONFIG_DIR "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}")

    # Make package configuration files
    # Use configure_package_config_file to make the package relocatable
    include(CMakePackageConfigHelpers)
    # Generate *Config.cmake file
    configure_package_config_file(
        "${CMAKE_CURRENT_LIST_DIR}/${PROJECT_NAME}Config.cmake.in"
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
        INSTALL_DESTINATION ${_PACKAGE_CONFIG_DIR}
    )
    # Generate *ConfigVersion.cmake file
    write_basic_package_version_file(
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
        VERSION ${PROJECT_VERSION}
        COMPATIBILITY AnyNewerVersion
    )
    # Install generated configuration files
    install(
        FILES "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
            "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
        DESTINATION ${_PACKAGE_CONFIG_DIR}
    )

    # Export targets
    # Make export target group
    install(
        EXPORT ${PROJECT_NAME}Targets
        FILE ${PROJECT_NAME}Targets.cmake
        NAMESPACE ${PROJECT_NAME}::
        DESTINATION ${_PACKAGE_CONFIG_DIR}
    )
    # Generate *Target.cmake file
    export(
        EXPORT ${PROJECT_NAME}Targets
        FILE ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Targets.cmake
        NAMESPACE ${PROJECT_NAME}::
    )
endif()

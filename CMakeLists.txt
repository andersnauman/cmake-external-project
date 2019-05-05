cmake_minimum_required(VERSION 3.14)
include(GNUInstallDirs)
include(ExternalProject)
project(window)

add_executable(${PROJECT_NAME} main.cpp)

set(CMAKE_INSTALL_PREFIX "install") # temp

# This function works around a CMake issue with setting include directories of
# imported libraries built with `ExternalProject_Add`.
# https://gitlab.kitware.com/cmake/cmake/issues/15052
function(MAKE_INCLUDE_DIR TARGET)
  get_target_property(DIR ${TARGET} INTERFACE_INCLUDE_DIRECTORIES)
  file(MAKE_DIRECTORY ${DIR})
endfunction()

# Dependencies
## SDL2
ExternalProject_Add(
    project_sdl
    HG_REPOSITORY "http://hg.libsdl.org/SDL"
    HG_TAG "release-2.0.9"
    PREFIX "${CMAKE_CURRENT_BINARY_DIR}/SDL"
    CMAKE_ARGS -DSDL_STATIC=OFF
    INSTALL_COMMAND ""
    UPDATE_COMMAND ""
)
ExternalProject_Get_Property(project_sdl SOURCE_DIR)
ExternalProject_Get_Property(project_sdl BINARY_DIR)

## SDL2 - Create a shared library
add_library(externallib SHARED IMPORTED)
set_target_properties(externallib PROPERTIES 
    IMPORTED_LOCATION 
        ${BINARY_DIR}/${CMAKE_SHARED_LIBRARY_PREFIX}SDL2-2.0${CMAKE_SHARED_LIBRARY_SUFFIX}
    INTERFACE_INCLUDE_DIRECTORIES
        ${SOURCE_DIR}/include
)
target_link_libraries(${PROJECT_NAME} externallib)

# ExternalProject_Add + INTERFACE_INCLUDE_DIRECTORIES != <3.
# https://gitlab.kitware.com/cmake/cmake/issues/15052
file(MAKE_DIRECTORY ${SOURCE_DIR}/include)

# Only so we can run the program in "build"-mode
add_custom_command(
        TARGET ${PROJECT_NAME} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy
            ${BINARY_DIR}/${CMAKE_SHARED_LIBRARY_PREFIX}SDL2-2.0${CMAKE_SHARED_LIBRARY_SUFFIX}
            ${CMAKE_SHARED_LIBRARY_PREFIX}SDL2-2.0${CMAKE_SHARED_LIBRARY_SUFFIX}
)

## SDL2 - Create static library for the generic main-function
add_library(staticlib STATIC IMPORTED)
set_target_properties(staticlib PROPERTIES IMPORTED_LOCATION ${BINARY_DIR}/${CMAKE_STATIC_LIBRARY_PREFIX}SDL2main${CMAKE_STATIC_LIBRARY_SUFFIX})
target_link_libraries(${PROJECT_NAME} staticlib)

add_dependencies(${PROJECT_NAME} project_sdl)
#target_link_directories(${PROJECT_NAME} INTERFACE $<BUILD_INTERFACE:${BINARY_DIR}> $<INSTALL_INTERFACE:${CMAKE_INSTALL_PREFIX}>)
set_target_properties(${PROJECT_NAME} PROPERTIES
    BUILD_RPATH
        "@loader_path"
    CMAKE_INSTALL_RPATH_USE_LINK_PATH
        TRUE
)
# ExternalProject_Add do not install files automaticly
install(
    FILES
        "${BINARY_DIR}/${CMAKE_SHARED_LIBRARY_PREFIX}SDL2-2.0${CMAKE_SHARED_LIBRARY_SUFFIX}"
    DESTINATION
        bin
)
install(
    TARGETS
        ${PROJECT_NAME}
    RUNTIME DESTINATION
        bin
    LIBRARY DESTINATION
        lib
    ARCHIVE DESTINATION
        lib    
)
cmake_minimum_required(VERSION 3.10)
project(discord-game-sdk-godot)

option(BUILD_SHARED_LIBS "Build shared libs" ON)

set(DISCORD_SDK_PATH "lib/discord-sdk" CACHE STRING "Discord SDK path")

###############################################################################
# conan stuff
###############################################################################

# export godot-cpp into local conan cache
execute_process(COMMAND conan export ${CMAKE_CURRENT_SOURCE_DIR}/lib/godot-cpp local/stable)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_BINARY_DIR})
list(APPEND CMAKE_PREFIX_PATH ${CMAKE_BINARY_DIR})

if(NOT EXISTS "${CMAKE_BINARY_DIR}/conan.cmake")
  message(STATUS "Downloading conan.cmake from https://github.com/conan-io/cmake-conan")
  file(DOWNLOAD "https://raw.githubusercontent.com/conan-io/cmake-conan/0.17.0/conan.cmake"
                "${CMAKE_BINARY_DIR}/conan.cmake"
                EXPECTED_HASH SHA256=3bef79da16c2e031dc429e1dac87a08b9226418b300ce004cc125a82687baeef
                TLS_VERIFY ON)
endif()

include(${CMAKE_BINARY_DIR}/conan.cmake)

conan_cmake_configure(REQUIRES godot-cpp/3.3@local/stable GENERATORS cmake_find_package)

conan_cmake_autodetect(settings)

conan_cmake_install(PATH_OR_REFERENCE . BUILD missing SETTINGS ${settings})

find_package(godot-cpp)

###############################################################################

# install build into example project by default
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
  set(
    CMAKE_INSTALL_PREFIX "../app/addons/discord_game_sdk" 
    CACHE PATH "..." 
    FORCE)
endif()

# let's us link relative to process on linux
set(CMAKE_INSTALL_RPATH "$ORIGIN")

file(GLOB_RECURSE DISCORD_SOURCES ${DISCORD_SDK_PATH}/cpp/*.cpp)

add_library(discord-game-sdk-godot src/Godot.cpp ${DISCORD_SOURCES})

set_target_properties(
  discord-game-sdk-godot
  PROPERTIES
  CXX_STANDARD 17
  CXX_STANDARD_REQUIRED ON)

target_include_directories(
  discord-game-sdk-godot 
  PUBLIC 
  ${DISCORD_SDK_PATH}/cpp)

if(APPLE)
  #set_target_properties(
  #  discord-game-sdk-godot
  #  PROPERTIES 
  #  OSX_ARCHITECTURES "arm64;x86_64")

  find_library(
    DISCORD_SDK 
    NAMES discord_game_sdk.dylib
    REQUIRED 
    PATHS ${DISCORD_SDK_PATH}/lib/x86_64)
else()
  find_library(
    DISCORD_SDK 
    NAMES libdiscord_game_sdk.so discord_game_sdk.dll
    REQUIRED 
    PATHS ${DISCORD_SDK_PATH}/lib/x86_64)
endif()

target_link_libraries(
  discord-game-sdk-godot
  PRIVATE
    godot-cpp::godot-cpp
    ${DISCORD_SDK})

install(TARGETS discord-game-sdk-godot DESTINATION .)

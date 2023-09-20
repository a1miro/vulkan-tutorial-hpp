cmake_minimum_required(VERSION 3.22)

if(DEFINED ENV{VCPKG_ROOT})
  message(STATUS "Found VCPKG_ROOT=$ENV{VCPKG_ROOT}")
  set(vcpkg_root "$ENV{VCPKG_ROOT}")
  set(CMAKE_TOOLCHAIN_FILE ${vcpkg_root}/scripts/buildsystems/vcpkg.cmake CACHE FILEPATH "" FORCE)
else()
  message(
    [[
    VCPKG_ROOT was not found,
    I highly recommend to install VCPKG and set VCPKG_ROOT, it can be done for Winows, MacOS and Linux.
    Please follow isntructions from this page https://vcpkg.io/en/getting-started.html,
    overwise you have to install manually the following packages:

    glm         OpenGL Mathematics
    glfw3       Free, open-source GL Window system
    stb         header only, portable C utility library
    ]])
endif()

project(VulkanTutorial LANGUAGES CXX)


if(DEFINED ENV{VULKAN_SDK})
  set(VULKAN_SDK "$ENV{VULKAN_SDK}")
  find_package(Vulkan REQUIRED)
else()
  message(STATUS "VULKAN_SDK variable was not found, trying to find it in the system")
  find_package(Vulkan REQUIRED)
  if(Vulkan_FOUND)
    message(STATUS "Vulkan SDK is found")
  else()
    message(FATAL_ERROR "Vulkan SDK is not found")
  endif()
endif()

add_compile_definitions(HOST_SYSTEM_NAME="${CMAKE_HOST_SYSTEM_NAME}")


find_program(GLSLC NAMES glslc glslc.exe)
if(GLSLC)
  message(STATUS "Found GLSLC compiler ${GLSLC}")
else()
  message(FATAL_ERROR "GSLC compiler is not found")
endif()

find_package(glfw3 REQUIRED)
find_package(glm REQUIRED)
find_package(Stb REQUIRED)


# retrieving all shader files from shaders sub-folder
file(GLOB_RECURSE SHADER_SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/shaders/*.frag ${CMAKE_CURRENT_SOURCE_DIR}/shaders/*.vert)

# retrieving all texture files from textures sub-folder
file(GLOB_RECURSE TEXTURE_FILES ${CMAKE_CURRENT_SOURCE_DIR}/textures/*)

# searching for all source files in current directory
file(GLOB_RECURSE CXX_SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/*.cpp)

# searching for all header files in current directory
file(GLOB_RECURSE CXX_HEADERS ${CMAKE_CURRENT_SOURCE_DIR}/*.h)

# retrieving base directory as project name
get_filename_component(projectId ${CMAKE_CURRENT_SOURCE_DIR} NAME)

add_executable(${projectId} ${CXX_SOURCES} ${CXX_HEADERS} ${SHADER_SOURCES})
target_link_libraries(${projectId} glm::glm Vulkan::Vulkan glfw)
target_include_directories(${projectId} PRIVATE ${CXX_HEADERS})
target_include_directories(${projectId} PRIVATE ${glfw_INCLUDE_DIRS})
target_include_directories(${projectId} PRIVATE ${Stb_INCLUDE_DIR})
target_include_directories(${projectId} PRIVATE ${Vulkan_INCLUDE_DIR})
set_property(TARGET ${projectId} PROPERTY CXX_STANDARD 17)

# Adding compiling shader command for every shader from SHADER_SRC
# the binary files will be put under output binary folder in shaders sub-folder
add_custom_command(
    TARGET ${projectId}
    PRE_BUILD
    COMMAND ${CMAKE_COMMAND}
    ARGS -E make_directory  "$<TARGET_FILE_DIR:${projectId}>/shaders"
    COMMAND ${CMAKE_COMMAND}
    ARGS -E make_directory  "$<TARGET_FILE_DIR:${projectId}>/textures"
  )

foreach(shader ${SHADER_SRC})
  cmake_path(GET shader EXTENSION LAST_ONLY shader_ext)
  string(REPLACE "." "" shader_ext ${shader_ext})
  add_custom_command(
    TARGET ${projectId}
    POST_BUILD
    COMMAND ${GLSLC}
    ARGS ${shader} -o "$<TARGET_FILE_DIR:${projectId}>/shaders/${shader_ext}.spv"
    DEPENDS ${shader}
  )
endforeach()

foreach(TEXTURE_FILE ${TEXTURE_FILES})
add_custom_command(
    TARGET ${projectId}
    POST_BUILD
    COMMAND ${CMAKE_COMMAND}
    ARGS -E copy  ${TEXTURE_FILE} "$<TARGET_FILE_DIR:${projectId}>/textures"
    DEPENDS ${TEXTURE_FILE}
  )
endforeach()

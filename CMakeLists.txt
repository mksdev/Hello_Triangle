cmake_minimum_required( VERSION 3.14.1 )
project( HelloTriangle )

# New&Nice cmake

find_package(Vulkan)
message(STATUS "Vulkan_FOUND : ${Vulkan_FOUND}")
message(STATUS "Vulkan_INCLUDE_DIRS : ${Vulkan_INCLUDE_DIRS}")
message(STATUS "Vulkan_LIBRARIES : ${Vulkan_LIBRARIES}")
message(STATUS "Vulkan_GLSLC_EXECUTABLE : ${Vulkan_GLSLC_EXECUTABLE}")


# Old&Ugly cmake

set( TODO ON CACHE BOOL "Enable compiletime TODO messages" )

# Select WSI platform (can use cmake -D)
set( WSI "USE_PLATFORM_GLFW" CACHE STRING "WSI type used by this app" )
message(STATUS  "WSI: " ${WSI} )

# Compile shaders
set( GLSL_DEBUG_FLAG $<$<CONFIG:Debug>:-g> )
set( GLSL_COMPILER ${Vulkan_GLSLC_EXECUTABLE} -mfmt=num ${GLSL_DEBUG_FLAG} )
message(STATUS "GLSL_COMPILER : ${GLSL_COMPILER}")

# Automatic find shaders to compile

# find all vertex shaders
file(GLOB_RECURSE SHADER_FILES FILES_MATCHING PATTERN "${CMAKE_CURRENT_SOURCE_DIR}/src/shaders/*.vert")
# find all fragment shaders
file(GLOB_RECURSE FRAGMENT_FILES FILES_MATCHING PATTERN "${CMAKE_CURRENT_SOURCE_DIR}/src/shaders/*.frag")
# .vert -> .spv.inl
string(REGEX REPLACE "[.]vert" ".spv.inl" SHADER_FILES_INCLUDE ${SHADER_FILES})
# .frag -> .spv.inl
string(REGEX REPLACE "[.]frag" ".spv.inl" FRAGMENT_FILES_INCLUDE ${FRAGMENT_FILES})

message(STATUS "SHADER_FILES: ${SHADER_FILES}")
message(STATUS "SHADER_FILES_INCLUDE: ${SHADER_FILES_INCLUDE}")

message(STATUS "FRAGMENT_FILES: ${FRAGMENT_FILES}")
message(STATUS "FRAGMENT_FILES_INCLUDE: ${FRAGMENT_FILES_INCLUDE}")

set(VERT_SHADER "${CMAKE_SOURCE_DIR}/src/shaders/hello_triangle.vert")
set(VERT_SHADER_INCLUDE ${VERT_SHADER}.spv.inl)
add_custom_command(
	COMMENT "Compiling vertex shader"
	MAIN_DEPENDENCY ${VERT_SHADER}
	OUTPUT ${VERT_SHADER_INCLUDE}
	COMMAND ${GLSL_COMPILER} -o ${VERT_SHADER_INCLUDE} ${VERT_SHADER}
	#VERBATIM -- TODO breaks empty generator-expression
)

set(FRAG_SHADER "${CMAKE_SOURCE_DIR}/src/shaders/hello_triangle.frag")
set(FRAG_SHADER_INCLUDE ${FRAG_SHADER}.spv.inl)
add_custom_command(
	COMMENT "Compiling fragment shader"
	MAIN_DEPENDENCY ${FRAG_SHADER}
	OUTPUT ${FRAG_SHADER_INCLUDE}
	COMMAND ${GLSL_COMPILER} -o ${FRAG_SHADER_INCLUDE} ${FRAG_SHADER}
	#VERBATIM -- TODO breaks empty generator-expression
)

add_custom_target(
	HelloTriangle_shaders
	COMMENT "Compiling shaders"
	DEPENDS ${VERT_SHADER_INCLUDE} ${FRAG_SHADER_INCLUDE}
)

# Build GLFW
if( ${WSI} STREQUAL "USE_PLATFORM_GLFW" )
	set( GLFW_BUILD_DOCS OFF CACHE BOOL "" FORCE )
	set( GLFW_BUILD_TESTS OFF CACHE BOOL "" FORCE )
	set( GLFW_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE )

	if( CYGWIN )
		# Hack to make GLFW use Win32 WSI instead of X11
		# TODO: Might be cleaner to use cross-compiling
		unset( UNIX )
		set( WIN32 1 )
	endif()

	add_subdirectory( external/glfw )
endif()

file(GLOB_RECURSE SOURCE_FILES FILES_MATCHING PATTERN "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp")

# HelloTriangle binary
set_property( DIRECTORY PROPERTY VS_STARTUP_PROJECT HelloTriangle )
add_executable( HelloTriangle WIN32 ${SOURCE_FILES})
add_dependencies( HelloTriangle HelloTriangle_shaders)

if( NOT TODO )
	add_definitions( -DNO_TODO )
endif()

if( MSVC )
	target_compile_definitions( HelloTriangle PRIVATE $<$<CONFIG:Debug>:_CONSOLE> )
	set_target_properties( HelloTriangle PROPERTIES LINK_FLAGS_DEBUG "/SUBSYSTEM:CONSOLE" )
endif()


if( ${WSI} STREQUAL "USE_PLATFORM_GLFW" )
	set( WSI_LIBS glfw )
elseif( ${WSI} STREQUAL "VK_USE_PLATFORM_WIN32_KHR" )
	set( WSI_LIBS )
	add_definitions( -DUNICODE )
elseif( ${WSI} STREQUAL "VK_USE_PLATFORM_XLIB_KHR" )
	set( WSI_LIBS X11 )
elseif( ${WSI} STREQUAL "VK_USE_PLATFORM_XCB_KHR" )
	set( WSI_LIBS xcb xcb-keysyms )
elseif( ${WSI} STREQUAL "VK_USE_PLATFORM_WAYLAND_KHR" )
	set( WSI_LIBS wayland-client xkbcommon )
endif()
add_definitions( -D${WSI} )

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

target_link_libraries( HelloTriangle
		Vulkan::Vulkan
		"${WSI_LIBS}" )

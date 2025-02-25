include(${CMAKE_ROOT}/Modules/ExternalProject.cmake)

# Make option() in all external projects do nothing when a normal variable of the
# same name exists. The normal variable is not removed. The cache entry is not
# created or updated and is ignored if it exists.
set(CMAKE_POLICY_DEFAULT_CMP0077 "NEW")

function(check_submodules_present)
	file(READ "${CMAKE_SOURCE_DIR}/.gitmodules" gitmodules)
	string(REGEX MATCHALL "path *= *[^ \t\r\n]*" gitmodules ${gitmodules})

	foreach(module ${gitmodules})
		# Get module name
		string(REGEX REPLACE "path *= *" "" module ${module})

		# Stat the folder and get ammount of entries
		file(GLOB RESULT "${CMAKE_SOURCE_DIR}/${module}/*")
		list(LENGTH RESULT RES_LEN)

		# If the module has no files, bring fatal error
		if(RES_LEN EQUAL 0)
			# directory is empty
			message(FATAL_ERROR "Submodule ${module} is empty. "
				"Please run: git submodule update --init --recursive")
		endif()
	endforeach()
endfunction()

check_submodules_present()

# Fix a glslang hack
# See https://github.com/KhronosGroup/glslang/issues/1015 and https://github.com/Vita3K/Vita3K/pull/369 for details
if(NOT CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
	if(WIN32)
		set(CMAKE_INSTALL_PREFIX "C:/Program Files/${PROJECT_NAME}" CACHE STRING "Default install prefix is: C:/Program Files" FORCE)
	elseif(UNIX)
		set(CMAKE_INSTALL_PREFIX /usr/local CACHE STRING "Default install prefix is: /usr/local" FORCE)
	endif()
endif()

option(CAPSTONE_BUILD_SHARED "Build shared library" OFF)
option(CAPSTONE_BUILD_TESTS "Build tests" OFF)
option(CAPSTONE_BUILD_CSTOOL "Build cstool" OFF)
option(CAPSTONE_ARCHITECTURE_DEFAULT "Whether architectures are enabled by default" OFF)
option(CAPSTONE_ARM_SUPPORT "ARM support" ON)
add_subdirectory(capstone EXCLUDE_FROM_ALL)

add_library(crypto-algorithms STATIC crypto-algorithms/sha256.c crypto-algorithms/sha256.h)
target_include_directories(crypto-algorithms PUBLIC "${CMAKE_CURRENT_SOURCE_DIR}/crypto-algorithms")
set_property(TARGET crypto-algorithms PROPERTY FOLDER externals)

add_library(dirent INTERFACE)

if(WIN32)
	target_include_directories(dirent INTERFACE "${CMAKE_CURRENT_SOURCE_DIR}/dirent/include")
endif()

add_library(printf INTERFACE)
target_include_directories(printf INTERFACE "${CMAKE_CURRENT_SOURCE_DIR}/printf")

add_subdirectory(fmt)
add_library(fmt::fmt ALIAS fmt)
set(fmt_DIR fmt)
set_property(TARGET fmt PROPERTY FOLDER externals)

set(SPDLOG_WCHAR_FILENAMES ON CACHE BOOL "")
set(SPDLOG_NO_THREAD_ID ON CACHE BOOL "")
set(SPDLOG_FMT_EXTERNAL ON CACHE BOOL "")
add_subdirectory(spdlog EXCLUDE_FROM_ALL)
set_property(TARGET spdlog PROPERTY FOLDER externals)

add_library(stb INTERFACE)
target_include_directories(stb INTERFACE "${CMAKE_CURRENT_SOURCE_DIR}/stb")

add_library(glad STATIC "${CMAKE_CURRENT_SOURCE_DIR}/glad/src/glad.c")
target_include_directories(glad PUBLIC "${CMAKE_CURRENT_SOURCE_DIR}/glad/include")
target_link_libraries(glad PRIVATE ${CMAKE_DL_LIBS})
set_property(TARGET glad PROPERTY FOLDER externals)

add_library(better-enums INTERFACE)
target_include_directories(better-enums INTERFACE "${CMAKE_CURRENT_SOURCE_DIR}/better-enums")
target_compile_definitions(better-enums INTERFACE BETTER_ENUMS_STRICT_CONVERSION=1)

add_library(googletest STATIC googletest/googletest/src/gtest_main.cc googletest/googletest/src/gtest-all.cc)
target_include_directories(googletest PUBLIC googletest/googletest/include)
target_include_directories(googletest PRIVATE googletest/googletest)
target_compile_definitions(googletest PUBLIC GTEST_HAS_PTHREAD=0)
set_property(TARGET googletest PROPERTY FOLDER externals)

add_subdirectory(libfat16)
set_property(TARGET FAT16 PROPERTY FOLDER externals)

# The imgui target is including both imgui and imgui_club.
add_library(imgui STATIC imgui/imgui.cpp imgui/imgui_draw.cpp imgui/imgui_tables.cpp imgui/imgui_widgets.cpp imgui/misc/cpp/imgui_stdlib.cpp)
target_compile_definitions(imgui PRIVATE IMGUI_DISABLE_DEMO_WINDOWS)
target_include_directories(imgui PUBLIC "${CMAKE_CURRENT_SOURCE_DIR}/imgui"
	"${CMAKE_CURRENT_SOURCE_DIR}/imgui_club/imgui_memory_editor/")
set_property(TARGET imgui PROPERTY FOLDER externals)

add_library(miniz STATIC miniz/miniz.c miniz/miniz.h)
target_include_directories(miniz PUBLIC "${CMAKE_CURRENT_SOURCE_DIR}/miniz")
set_property(TARGET miniz PROPERTY FOLDER externals)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/sdl2-cmake-scripts")

if(APPLE)
	set(SDL2_PATH "${CMAKE_CURRENT_SOURCE_DIR}/sdl/macos" CACHE PATH "Where SDL2 is located" FORCE)
elseif(WIN32)
	set(SDL2_PATH "${CMAKE_CURRENT_SOURCE_DIR}/sdl/windows" CACHE PATH "Where SDL2 is located" FORCE)
	set(SDL2_INCLUDE_DIR "${SDL2_PATH}/include" CACHE PATH "Where the SDL2 include files can be found" FORCE)
endif()

find_package(SDL2 REQUIRED)
add_library(sdl2 INTERFACE)
target_include_directories(sdl2 INTERFACE "${SDL2_INCLUDE_DIR}")
target_link_libraries(sdl2 INTERFACE "${SDL2_LIBRARY}")

# Cubeb setup, from https://github.com/RPCS3/rpcs3/blob/master/3rdparty/cubeb/CMakeLists.txt
set(BUILD_SHARED_LIBS FALSE CACHE BOOL "Don't build shared libs")
set(BUILD_TESTS FALSE CACHE BOOL "Don't build tests")
set(BUILD_RUST_LIBS FALSE CACHE BOOL "Don't build rust libs")
set(BUILD_TOOLS FALSE CACHE BOOL "Don't build tools")
set(BUNDLE_SPEEX TRUE CACHE BOOL "Bundle the speex library")
set(LAZY_LOAD_LIBS TRUE CACHE BOOL "Lazily load shared libraries")
set(USE_SANITIZERS FALSE CACHE BOOL "Dont't use sanitizers")

add_subdirectory(cubeb EXCLUDE_FROM_ALL)
set_property(TARGET cubeb PROPERTY FOLDER externals)
set_property(TARGET speex PROPERTY FOLDER externals)

if (CMAKE_SYSTEM_PROCESSOR MATCHES "^(arm|ARM|aarch64|AArch64|Aarch64)")
	target_compile_definitions(speex PUBLIC
		#_USE_NEON
	)
elseif (CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86|X86|amd64|AMD64|em64t|EM64T)")
	target_compile_definitions(speex PUBLIC
		_USE_SSE
		_USE_SSE2
	)
endif ()

if(WIN32)
	add_library(winsock INTERFACE)
	find_library(WSOCK32 wsock32)
	find_library(WS2_32 ws2_32)
	find_library(IPHLPAPI iphlpapi)
	target_link_libraries(winsock INTERFACE WSOCK32 WS2_32 IPHLPAPI)
endif()

if(WIN32)
	set(UNICORN_VER "4b52942b")

	set(UNICORN_PREFIX "${CMAKE_BINARY_DIR}/external/unicorn")

	if(NOT EXISTS "${UNICORN_PREFIX}")
		message(STATUS "Downloading unicorn...")

		file(DOWNLOAD
			https://github.com/Vita3K/unicorn/releases/download/${UNICORN_VER}/vita3k-unicorn.zip
			"${CMAKE_BINARY_DIR}/external/unicorn.zip" SHOW_PROGRESS)
		execute_process(COMMAND ${CMAKE_COMMAND} -E tar xf "${CMAKE_BINARY_DIR}/external/unicorn.zip"
			WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/external")
	endif()

	message(STATUS "Using bundled binaries at ${UNICORN_PREFIX}")

	set(LIBUNICORN_INCLUDE_DIR "${UNICORN_PREFIX}/include" CACHE PATH "Path to Unicorn headers" FORCE)
	set(LIBUNICORN_LIBRARY "${UNICORN_PREFIX}/lib/unicorn.lib" CACHE PATH "Path to Unicorn library" FORCE)
	set(UNICORN_DLL_DIR "${UNICORN_PREFIX}/lib" CACHE PATH "Path to unicorn.dll" FORCE)

	add_library(unicorn INTERFACE)
	target_link_libraries(unicorn INTERFACE "${LIBUNICORN_LIBRARY}")
	target_include_directories(unicorn INTERFACE "${LIBUNICORN_INCLUDE_DIR}")
else()
	option(UNICORN_BUILD_SHARED "Build shared instead of static library" OFF)
	set(UNICORN_ARCH "arm")
	add_subdirectory(unicorn)
	target_include_directories(unicorn INTERFACE "${CMAKE_CURRENT_SOURCE_DIR}/unicorn/include")
endif()

set(DYNARMIC_TESTS OFF CACHE BOOL "")
set(DYNARMIC_NO_BUNDLED_FMT ON CACHE BOOL "")
set(DYNARMIC_FRONTENDS "A32" CACHE STRING "")
add_subdirectory(dynarmic)
set_property(TARGET dynarmic PROPERTY FOLDER externals)
set_property(TARGET Zycore PROPERTY FOLDER externals)
set_property(TARGET Zydis PROPERTY FOLDER externals)
set_property(TARGET mcl PROPERTY FOLDER externals)

if(MSVC)
    if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
        list(APPEND MCL_CXX_FLAGS
             -Qunused-arguments
             -Wno-missing-braces)
		target_compile_options(mcl PRIVATE ${MCL_CXX_FLAGS})
    endif()
endif()

add_library(vita-toolchain INTERFACE)
target_include_directories(vita-toolchain INTERFACE "${CMAKE_CURRENT_SOURCE_DIR}/vita-toolchain/src")

option(BUILD_TESTING "Build the testing tree." OFF)
option(YAML_CPP_BUILD_TOOLS "Enable parse tools" OFF)
option(YAML_CPP_BUILD_CONTRIB "Enable contrib stuff in library" OFF)
option(YAML_CPP_INSTALL "Enable generation of install target" OFF)
add_subdirectory(yaml-cpp EXCLUDE_FROM_ALL)
set_property(TARGET yaml-cpp PROPERTY FOLDER externals)

if(USE_DISCORD_RICH_PRESENCE)
	if(NOT EXISTS "${CMAKE_BINARY_DIR}/external/discord_game_sdk.zip")
		message(STATUS "Downloading discord gamesdk...")
		file(DOWNLOAD https://dl-game-sdk.discordapp.net/2.5.6/discord_game_sdk.zip
			"${CMAKE_BINARY_DIR}/external/discord_game_sdk.zip" SHOW_PROGRESS)
	endif()
endif()

if(USE_DISCORD_RICH_PRESENCE)
	if(NOT EXISTS "${CMAKE_BINARY_DIR}/external/discord_game_sdk")
		file(MAKE_DIRECTORY "${CMAKE_BINARY_DIR}/external/discord_game_sdk")
		execute_process(COMMAND ${CMAKE_COMMAND} -E tar xf "${CMAKE_BINARY_DIR}/external/discord_game_sdk.zip"
			WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/external/discord_game_sdk")
		file(RENAME "${CMAKE_BINARY_DIR}/external/discord_game_sdk/lib/x86_64/discord_game_sdk.so"
			"${CMAKE_BINARY_DIR}/external/discord_game_sdk/lib/x86_64/libdiscord_game_sdk.so")
	endif()

	add_library(discord-rpc STATIC
		${CMAKE_BINARY_DIR}/external/discord_game_sdk/cpp/achievement_manager.cpp
		${CMAKE_BINARY_DIR}/external/discord_game_sdk/cpp/activity_manager.cpp
		${CMAKE_BINARY_DIR}/external/discord_game_sdk/cpp/application_manager.cpp
		${CMAKE_BINARY_DIR}/external/discord_game_sdk/cpp/core.cpp
		${CMAKE_BINARY_DIR}/external/discord_game_sdk/cpp/image_manager.cpp
		${CMAKE_BINARY_DIR}/external/discord_game_sdk/cpp/lobby_manager.cpp
		${CMAKE_BINARY_DIR}/external/discord_game_sdk/cpp/network_manager.cpp
		${CMAKE_BINARY_DIR}/external/discord_game_sdk/cpp/overlay_manager.cpp
		${CMAKE_BINARY_DIR}/external/discord_game_sdk/cpp/relationship_manager.cpp
		${CMAKE_BINARY_DIR}/external/discord_game_sdk/cpp/storage_manager.cpp
		${CMAKE_BINARY_DIR}/external/discord_game_sdk/cpp/store_manager.cpp
		${CMAKE_BINARY_DIR}/external/discord_game_sdk/cpp/types.cpp
		${CMAKE_BINARY_DIR}/external/discord_game_sdk/cpp/user_manager.cpp
		${CMAKE_BINARY_DIR}/external/discord_game_sdk/cpp/voice_manager.cpp)

	if(APPLE)
		target_link_libraries(discord-rpc PUBLIC "${CMAKE_BINARY_DIR}/external/discord_game_sdk/lib/x86_64/discord_game_sdk.dylib")
	elseif(WIN32)
		target_link_libraries(discord-rpc PUBLIC "${CMAKE_BINARY_DIR}/external/discord_game_sdk/lib/x86_64/discord_game_sdk.dll.lib")
	elseif(UNIX)
		target_link_libraries(discord-rpc PUBLIC "${CMAKE_BINARY_DIR}/external/discord_game_sdk/lib/x86_64/libdiscord_game_sdk.so")
	endif()

	target_include_directories(discord-rpc PUBLIC "${CMAKE_BINARY_DIR}/external/discord_game_sdk/cpp")
	set_property(TARGET discord-rpc PROPERTY FOLDER externals)
endif()

option(BUILD_EXTERNAL "Build external dependencies in /External" OFF)
option(SKIP_GLSLANG_INSTALL "Skip installation" ON)
option(ENABLE_SPVREMAPPER "Enables building of SPVRemapper" OFF)
option(ENABLE_GLSLANG_BINARIES "Builds glslangValidator and spirv-remap" OFF)
option(ENABLE_HLSL "Enables HLSL input support" OFF)
option(ENABLE_CTEST "Enables testing" OFF)
add_subdirectory(glslang)

option(SPIRV_CROSS_CLI "Build the CLI binary. Requires SPIRV_CROSS_STATIC." OFF)
option(SPIRV_CROSS_ENABLE_TESTS "Enable SPIRV-Cross tests." OFF)
option(SPIRV_CROSS_ENABLE_HLSL "Enable HLSL target support." OFF)
option(SPIRV_CROSS_ENABLE_MSL "Enable MSL target support." OFF)
option(SPIRV_CROSS_ENABLE_CPP "Enable C++ target support." OFF)
option(SPIRV_CROSS_ENABLE_REFLECT "Enable JSON reflection target support." OFF)
option(SPIRV_CROSS_ENABLE_C_API "Enable C API wrapper support in static library." OFF)
option(SPIRV_CROSS_ENABLE_UTIL "Enable util module support." OFF)
option(SPIRV_CROSS_SKIP_INSTALL "Skips installation targets." ON)
add_subdirectory(SPIRV-Cross)
set_property(TARGET spirv-cross-core PROPERTY FOLDER externals)
set_property(TARGET spirv-cross-glsl PROPERTY FOLDER externals)

add_library(dlmalloc STATIC "${CMAKE_CURRENT_SOURCE_DIR}/dlmalloc/dlmalloc.cc")
target_include_directories(dlmalloc PUBLIC "${CMAKE_CURRENT_SOURCE_DIR}/dlmalloc")
target_compile_definitions(dlmalloc PUBLIC ONLY_MSPACES=1)
target_compile_definitions(dlmalloc PUBLIC USE_LOCK=0)
set_property(TARGET dlmalloc PROPERTY FOLDER externals)

add_subdirectory(pugixml)
set_property(TARGET pugixml-static PROPERTY FOLDER externals)

add_library(CLI11 INTERFACE)

# See "Note: Special instructions for GCC 8" on https://github.com/CLIUtils/CLI11
if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU" AND CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 8 AND CMAKE_CXX_COMPILER_VERSION VERSION_LESS 9)
	target_compile_definitions(CLI11 INTERFACE CLI11_HAS_FILESYSTEM=0)
endif()

target_include_directories(CLI11 INTERFACE "${CMAKE_CURRENT_SOURCE_DIR}/cli11")

add_library(vulkan INTERFACE)
target_include_directories(vulkan INTERFACE "${CMAKE_CURRENT_SOURCE_DIR}/Vulkan-Headers/include")

add_library(vma INTERFACE)
target_include_directories(vma INTERFACE
	"${CMAKE_CURRENT_SOURCE_DIR}/VulkanMemoryAllocator-Hpp/include"
	"${CMAKE_CURRENT_SOURCE_DIR}/VulkanMemoryAllocator-Hpp/VulkanMemoryAllocator/include")

add_subdirectory(ffmpeg)
add_subdirectory(psvpfstools)
set_property(TARGET psvpfsparser PROPERTY FOLDER externals)
set_property(TARGET libzRIF PROPERTY FOLDER externals)
set_property(TARGET libb64 PROPERTY FOLDER externals)

if(APPLE)
	set(OPENSSL_ROOT_DIR "/usr/local/opt/openssl")
	set(OPENSSL_LIBRARIES "/usr/local/opt/openssl/lib")
	set(OPENSSL_USE_STATIC_LIBS TRUE)
endif()
find_package(OpenSSL QUIET)

if(NOT OPENSSL_FOUND)
	if(MSVC)
		message("OpenSSL not found, using prebuilt version")

		if(NOT EXISTS "${CMAKE_BINARY_DIR}/external/openssl.zip")
			message(STATUS "Downloading openssl...")
			file(DOWNLOAD https://firedaemon.com/download-firedaemon-openssl-3-zip
				"${CMAKE_BINARY_DIR}/external/openssl.zip" SHOW_PROGRESS)
		endif()

		if(NOT EXISTS "${CMAKE_BINARY_DIR}/external/openssl")
			file(MAKE_DIRECTORY "${CMAKE_BINARY_DIR}/external/openssl")
			execute_process(COMMAND ${CMAKE_COMMAND} -E tar xf "${CMAKE_BINARY_DIR}/external/openssl.zip"
				WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/external/openssl")
		endif()

		set(OPENSSL_ROOT_DIR "${CMAKE_BINARY_DIR}/external/openssl/openssl-3/x64")
	endif()

	find_package(OpenSSL REQUIRED)
endif()

message("OpenSSL include dir: ${OPENSSL_INCLUDE_DIR}")
message("OpenSSL libraries: ${OPENSSL_LIBRARIES}")

add_library(ssl INTERFACE)
target_link_libraries(ssl INTERFACE OpenSSL::SSL)

file(GLOB LIBATRAC9_SOURCES
	LibAtrac9/C/src/*.c
	LibAtrac9/C/src/*.h
)
add_library(libatrac9 STATIC ${LIBATRAC9_SOURCES})

target_include_directories(libatrac9 PUBLIC LibAtrac9/C/src)
set_property(TARGET libatrac9 PROPERTY FOLDER externals)

option(BUILD_SHARED_LIBS "Build shared library" OFF)
option(XXHASH_BUILD_XXHSUM "Build the xxhsum binary" OFF)
add_subdirectory(xxHash/cmake_unofficial EXCLUDE_FROM_ALL)
set_property(TARGET xxhash PROPERTY FOLDER externals)

# Tracy
option(TRACY_ENABLE_ON_CORE_COMPONENTS
	"Enable and require Tracy to compile core components such as the renderer, shader recompiler and
		HLE modules"
	ON)
add_library(tracy STATIC tracy/public/TracyClient.cpp)
set_property(TARGET tracy PROPERTY FOLDER externals)
target_include_directories(tracy PUBLIC tracy)

if(UNIX)
	target_link_libraries(tracy PUBLIC pthread dl)
endif()

# Enable Tracy on-demand profiling mode to avoid unnecesary memory usage when the developer
# isn't profiling by only enabling profiling after a connection has been established with
# a Tracy server
target_compile_definitions(tracy PUBLIC $<$<CONFIG:Debug,RelWithDebInfo>:TRACY_ON_DEMAND>)

# Disable Tracy automated data collection in order to prevent Tracy-related code from being profiled
target_compile_definitions(tracy PUBLIC TRACY_NO_SYSTEM_TRACING)

# Defining TRACY_ENABLE for both compiling Tracy and compiling the project that links
# against it is needed for Tracy to work
#
# -------------------------- Options (choose one at a time) ---------------------------
#
# A (DEFAULT). Enable Tracy on any target that links against if the build configuration
# is either Debug and RelWithDebInfo
#
# Using generator expresions instead of a conditional with CMAKE_BUILD_TYPE is needed
# in order for the condition to work properly on both single-config and multi-config
# CMake project generators. More info here:
# https://cmake.org/cmake/help/latest/manual/cmake-buildsystem.7.html#build-configurations
target_compile_definitions(tracy PUBLIC $<$<CONFIG:Debug,RelWithDebInfo>:TRACY_ENABLE>)

#
# ----------------------------------------------------------------------------------------
#
# B. Enable Tracy on every target that links to it regardless of the
# build configuration:
#
# target_compile_definitions(tracy PUBLIC TRACY_ENABLE)

# Use XDG desktop portals on Linux
set(NFD_PORTAL ON)

# nativefiledialog-extended
add_subdirectory(nativefiledialog-extended)

# Create alias to prevent the need of multiple changes in case the target name changes
add_library(NFDe::NFDe ALIAS nfd)
set_property(TARGET nfd PROPERTY FOLDER externals)

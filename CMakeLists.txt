########################################
# General setup
#
cmake_minimum_required(VERSION 2.8.8)
project(dolphin-emu)

option(USE_EGL "Enables EGL OpenGL Interface" OFF)
option(TRY_X11 "Enables X11 Support" ON)
option(USE_SHARED_ENET "Use shared libenet if found rather than Dolphin's soon-to-compatibly-diverge version" OFF)
option(USE_SHARED_GTEST "Use shared gtest library if found" OFF)
option(USE_UPNP "Enables UPnP port mapping support" ON)
option(DISABLE_WX "Disable wxWidgets (use CLI interface)" OFF)
option(ENABLE_PCH "Use PCH to speed up compilation" ON)
option(ENABLE_LTO "Enables Link Time Optimization" OFF)
option(ENABLE_GENERIC "Enables generic build that should run on any little-endian host" OFF)
option(ENABLE_HEADLESS "Enables running Dolphin as a headless variant" OFF)

# Maintainers: if you consider blanket disabling this for your users, please
# consider the following points:
#  * No data is being sent without explicit user approval (pop up box at first
#    launch).
#  * The Dolphin team relies on the data in order to understand the behavior
#    of our software in the wild.
option(ENABLE_ANALYTICS "Enables opt-in Analytics collection" ON)

# Name of the Dolphin distributor. If you redistribute Dolphin builds (forks,
# unofficial builds) please consider identifying your distribution with a
# unique name here.
set(DISTRIBUTOR "None" CACHE STRING "Name of the distributor.")

# Enable SDL for default on operating systems that aren't OSX, Android, Linux or Windows.
if(NOT APPLE AND NOT ANDROID AND NOT ${CMAKE_SYSTEM_NAME} STREQUAL "Linux" AND NOT MSVC)
	option(ENABLE_SDL "Enables SDL as a generic controller backend" ON)
else()
	option(ENABLE_SDL "Enables SDL as a generic controller backend" OFF)
endif()

if(${CMAKE_SYSTEM_NAME} STREQUAL "Linux" AND NOT ANDROID)
	option(ENABLE_EVDEV "Enables the evdev controller backend" ON)
endif()

if(APPLE)
	option(OSX_USE_DEFAULT_SEARCH_PATH "Don't prioritize system library paths" OFF)
endif()

option(ENCODE_FRAMEDUMPS "Encode framedumps in AVI format" ON)

option(FASTLOG "Enable all logs" OFF)
option(OPROFILING "Enable profiling" OFF)
option(GDBSTUB "Enable gdb stub for remote debugging." OFF)
if(${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
	option(VTUNE "Enable Intel VTune integration for JIT symbols." OFF)
endif()

if(APPLE)
	option(SKIP_POSTPROCESS_BUNDLE "Skip postprocessing bundle for redistributability" OFF)
endif()
########################################
# Optional Targets
# TODO: Add DSPSpy
option(DSPTOOL "Build dsptool" OFF)

# Update compiler before calling project()
if (APPLE)
	# Use clang compiler
	if (NOT DEFINED CMAKE_CXX_COMPILER)
		set(CMAKE_CXX_COMPILER "/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang++")
		if (NOT EXISTS "${CMAKE_CXX_COMPILER}")
			set(CMAKE_CXX_COMPILER "clang++")
		endif()
	endif()
	if (NOT DEFINED CMAKE_C_COMPILER)
		set(CMAKE_C_COMPILER "/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang")
		if (NOT EXISTS "${CMAKE_C_COMPILER}")
			set(CMAKE_C_COMPILER "clang")
		endif()
	endif()
endif()
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_SOURCE_DIR}/CMakeTests)
# Libraries to link
set(LIBS)

# Set up paths
if(APPLE)
	# The gettext module will install the translations unconditionally.
	# Redirect the installation to a build directory where it does no harm.
	set(CMAKE_INSTALL_PREFIX ${CMAKE_BINARY_DIR}/install-dummy)
else()
	set(bindir  ${CMAKE_INSTALL_PREFIX}/bin               CACHE PATH "bindir")
	set(datadir ${CMAKE_INSTALL_PREFIX}/share/dolphin-emu CACHE PATH "datadir")
	set(mandir  ${CMAKE_INSTALL_PREFIX}/share/man         CACHE PATH "mandir")
	add_definitions(-DDATA_DIR="${datadir}/")
endif()

# Set where the binary files will be built.  The program will not execute from
# here.  You must run "make install" to install these to the proper location
# as defined above.
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/Binaries)

# Precompiled header support for MSVC:
# Call this after setting the source list (and don't add the source file used
# to generate the pch file, this will be done here automatically)
function(enable_precompiled_headers PRECOMPILED_HEADER SOURCE_FILE SOURCE_VARIABLE_NAME)
	if(MSVC)
		set(files ${${SOURCE_VARIABLE_NAME}})

		# Generate precompiled header translation unit
		get_filename_component(pch_basename ${PRECOMPILED_HEADER} NAME_WE)
		set(pch_abs ${CMAKE_CURRENT_SOURCE_DIR}/${PRECOMPILED_HEADER})
		set(pch_unity ${CMAKE_CURRENT_SOURCE_DIR}/${SOURCE_FILE})
		set_source_files_properties(${pch_unity}  PROPERTIES COMPILE_FLAGS
			"/Yc\"${pch_abs}\"")

		# Update properties of source files to use the precompiled header.
		# Additionally, force the inclusion of the precompiled header at
		# beginning of each source file.
		foreach(source_file ${files} )
			set_source_files_properties(${source_file} PROPERTIES COMPILE_FLAGS
				"/Yu\"${pch_abs}\" /FI\"${pch_abs}\"")
		endforeach(source_file)

		# Finally, update the source file collection to contain the
		# precompiled header translation unit
		set(${SOURCE_VARIABLE_NAME} ${pch_unity} ${${SOURCE_VARIABLE_NAME}} PARENT_SCOPE)
	endif(MSVC)
endfunction(enable_precompiled_headers)

# for revision info
include(FindGit OPTIONAL)
if(GIT_FOUND AND NOT DOLPHIN_WC_REVISION)
	# defines DOLPHIN_WC_REVISION
	EXECUTE_PROCESS(WORKING_DIRECTORY ${PROJECT_SOURCE_DIR} COMMAND ${GIT_EXECUTABLE} rev-parse HEAD
		OUTPUT_VARIABLE DOLPHIN_WC_REVISION
		OUTPUT_STRIP_TRAILING_WHITESPACE)
	# defines DOLPHIN_WC_DESCRIBE
	EXECUTE_PROCESS(WORKING_DIRECTORY ${PROJECT_SOURCE_DIR} COMMAND ${GIT_EXECUTABLE} describe --always --long --dirty
		OUTPUT_VARIABLE DOLPHIN_WC_DESCRIBE
		OUTPUT_STRIP_TRAILING_WHITESPACE)

	# remove hash (and trailing "-0" if needed) from description
	STRING(REGEX REPLACE "(-0)?-[^-]+((-dirty)?)$" "\\2" DOLPHIN_WC_DESCRIBE "${DOLPHIN_WC_DESCRIBE}")

	# defines DOLPHIN_WC_BRANCH
	EXECUTE_PROCESS(WORKING_DIRECTORY ${PROJECT_SOURCE_DIR} COMMAND ${GIT_EXECUTABLE} rev-parse --abbrev-ref HEAD
		OUTPUT_VARIABLE DOLPHIN_WC_BRANCH
		OUTPUT_STRIP_TRAILING_WHITESPACE)
endif()

# version number
set(DOLPHIN_VERSION_MAJOR "5")
set(DOLPHIN_VERSION_MINOR "0")
if(DOLPHIN_WC_BRANCH STREQUAL "stable")
	set(DOLPHIN_VERSION_PATCH "0")
else()
	set(DOLPHIN_VERSION_PATCH ${DOLPHIN_WC_REVISION})
endif()

# If Dolphin is not built from a Git repository, default the version info to
# reasonable values.
if(NOT DOLPHIN_WC_REVISION)
	set(DOLPHIN_WC_DESCRIBE "${DOLPHIN_VERSION_MAJOR}.${DOLPHIN_VERSION_MINOR}")
	set(DOLPHIN_WC_REVISION "${DOLPHIN_WC_DESCRIBE} (no further info)")
	set(DOLPHIN_WC_BRANCH "master")
endif()

# Architecture detection and arch specific settings
message(${CMAKE_SYSTEM_PROCESSOR})

# Detect 64bit or 32bit
# CMake doesn't provide a simple way to determine 32bit or 64bit
# If we ever support a architecture that is 64bit with 32bit pointers then this'll break
# Of course the chances of that are slim(x32?) so who cares
if(CMAKE_SIZEOF_VOID_P EQUAL 8)
	set(_ARCH_64 1)
	add_definitions(-D_ARCH_64=1)
else()
	set(_ARCH_32 1)
	add_definitions(-D_ARCH_32=1)
endif()

include(CheckCCompilerFlag)
if(ENABLE_GENERIC)
	message("Warning! Building generic build!")
	set(_M_GENERIC 1)
	add_definitions(-D_M_GENERIC=1)
elseif(_ARCH_64 AND (
	${CMAKE_SYSTEM_PROCESSOR} MATCHES "^x86" OR
	${CMAKE_SYSTEM_PROCESSOR} MATCHES "i.86" OR
	${CMAKE_SYSTEM_PROCESSOR} MATCHES "amd64" OR
	APPLE
))
	set(_M_X86 1)
	set(_M_X86_64 1)
	add_definitions(-D_M_X86=1 -D_M_X86_64=1 -msse2)
	set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fno-pie")
	if(CMAKE_CXX_COMPILER_ID STREQUAL GNU)
		CHECK_C_COMPILER_FLAG("-no-pie" NO_PIE_UPSTREAM)
		if(NO_PIE_UPSTREAM)
			set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -no-pie")
		endif()
		CHECK_C_COMPILER_FLAG("-nopie" NO_PIE_PATCHED)
		if(NO_PIE_PATCHED)
			set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -nopie")
		endif()
	endif()
elseif(${CMAKE_SYSTEM_PROCESSOR} STREQUAL "aarch64")
	set(_M_ARM 1)
	set(_M_ARM_64 1)
	add_definitions(-D_M_ARM=1 -D_M_ARM_64=1)
	add_definitions(-march=armv8-a+crc)
else()
	message(FATAL_ERROR "You're building on an unsupported platform. Enable generic build if you really want a JIT-less binary.")
endif()

include(CheckCXXCompilerFlag)
macro(check_and_add_flag var flag)
	CHECK_CXX_COMPILER_FLAG(${flag} FLAG_${var})
	if(FLAG_${var})
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${flag}")
	endif()
endmacro()

# Enabling all warnings in MSVC spams too much
if(NOT MSVC)
	add_definitions(-Wall)
	add_definitions(-D_DEFAULT_SOURCE)
	# TODO: would like these but they produce overwhelming amounts of warnings
	#check_and_add_flag(EXTRA -Wextra)
	#check_and_add_flag(MISSING_FIELD_INITIALIZERS -Wmissing-field-initializers)
	#check_and_add_flag(SWITCH_DEFAULT -Wswitch-default)
	#check_and_add_flag(FLOAT_EQUAL -Wfloat-equal)
	#check_and_add_flag(CONVERSION -Wconversion)
	#check_and_add_flag(ZERO_AS_NULL_POINTER_CONSTANT -Wzero-as-null-pointer-constant)
	check_and_add_flag(TYPE_LIMITS -Wtype-limits)
	check_and_add_flag(SIGN_COMPARE -Wsign-compare)
	check_and_add_flag(IGNORED_QUALIFIERS -Wignored-qualifiers)
	check_and_add_flag(UNINITIALIZED -Wuninitialized)
	check_and_add_flag(LOGICAL_OP -Wlogical-op)
	check_and_add_flag(SHADOW -Wshadow)
	check_and_add_flag(INIT_SELF -Winit-self)
	check_and_add_flag(MISSING_DECLARATIONS -Wmissing-declarations)
	check_and_add_flag(MISSING_VARIABLE_DECLARATIONS -Wmissing-variable-declarations)
endif(NOT MSVC)

# gcc uses some optimizations which might break stuff without this flag
add_definitions(-fno-strict-aliasing -fno-exceptions)

check_and_add_flag(VISIBILITY_INLINES_HIDDEN -fvisibility-inlines-hidden)

if(UNIX AND NOT APPLE)
	check_and_add_flag(VISIBILITY_HIDDEN -fvisibility=hidden)
endif()

if(ENABLE_LTO)
	check_and_add_flag(LTO -flto)
	if(CMAKE_CXX_COMPILER_ID STREQUAL GNU)
		set(CMAKE_AR gcc-ar)
		set(CMAKE_RANLIB gcc-ranlib)
	endif()
endif()

if(APPLE)
	if(NOT OSX_USE_DEFAULT_SEARCH_PATH)
		# Hack up the path to prioritize the path to built-in OS libraries to
		# increase the chance of not depending on a bunch of copies of them
		# installed by MacPorts, Fink, Homebrew, etc, and ending up copying
		# them into the bundle.  Since we optionally depend on libraries which
		# are not part of OS X (ffmpeg, libusb, etc.), however, don't remove
		# the default path entirely as was done in a previous version of this
		# file.  This is still kinda evil, since it defeats the user's path
		# settings...
		# See http://www.cmake.org/cmake/help/v3.0/command/find_program.html
		set(CMAKE_PREFIX_PATH "${CMAKE_PREFIX_PATH};/usr")
	endif()

	# Identify the target system:
	# Ask for 64-bit binary.
	set(TARGET_FLAGS "-arch x86_64")
	# Minimum OS X version.
	# This is inserted into the Info.plist as well.
	# Note that the SDK determines the maximum version of which optional
	# features can be used, not the minimum required version to run.
	set(OSX_MIN_VERSION "10.9")
	set(TARGET_FLAGS "${TARGET_FLAGS} -mmacosx-version-min=${OSX_MIN_VERSION}")
	set(SYSROOT_LEGACY_PATH "/Developer/SDKs/MacOSX10.9.sdk")
	set(SYSROOT_PATH "/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.9.sdk")
	if(EXISTS "${SYSROOT_PATH}/")
		set(TARGET_SYSROOT ${SYSROOT_PATH})
	elseif(EXISTS "${SYSROOT_LEGACY_PATH}/")
		set(TARGET_SYSROOT ${SYSROOT_LEGACY_PATH})
	endif()
	if(${TARGET_SYSROOT})
		set(TARGET_FLAGS "${TARGET_FLAGS} -isysroot ${TARGET_SYSROOT}")
		set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,-syslibroot,${TARGET_SYSROOT}")
	endif()
	# Do not warn about frameworks that are not available on all architectures.
	# This avoids a warning when linking with QuickTime.
	set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,-no_arch_warnings")
	# Specify target CPUs.
	set(TARGET_FLAGS "${TARGET_FLAGS} -mssse3")
	set(TARGET_FLAGS "${TARGET_FLAGS} -march=core2")
	# Target flags apply to both C and C++ compilation.
	# CMake passes these to the compiler on the link command line as well.
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${TARGET_FLAGS}")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${TARGET_FLAGS}")

	# Linker flags.
	# Drop unreachable code and data.
	set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,-dead_strip,-dead_strip_dylibs")
	# Reserve the minimum size for the zero page.
	# Our JIT requires virtual memory space below 2GB, while the default zero
	# page on x86_64 is 4GB in size.
	set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,-pagezero_size,0x1000")

	find_library(APPKIT_LIBRARY AppKit)
	find_library(APPSERV_LIBRARY ApplicationServices)
	find_library(ATB_LIBRARY AudioToolbox)
	find_library(AU_LIBRARY AudioUnit)
	find_library(CARBON_LIBRARY Carbon)
	find_library(COCOA_LIBRARY Cocoa)
	find_library(COREAUDIO_LIBRARY CoreAudio)
	find_library(COREFUND_LIBRARY CoreFoundation)
	find_library(CORESERV_LIBRARY CoreServices)
	find_library(FOUNDATION_LIBRARY foundation)
	find_library(IOB_LIBRARY IOBluetooth)
	find_library(IOK_LIBRARY IOKit)
	find_library(QUICKTIME_LIBRARY QuickTime)
	find_library(WEBKIT_LIBRARY WebKit)
	find_library(FORCEFEEDBACK ForceFeedback)
	find_library(OPENGL_LIBRARY OpenGL)
endif()

if(WIN32)
	add_definitions(-D_SECURE_SCL=0)
	add_definitions(-D_CRT_SECURE_NO_WARNINGS)
	add_definitions(-D_CRT_SECURE_NO_DEPRECATE)
endif(WIN32)

# Add an option to build relocatable binaries on Linux
# The Sys folder will need to be copied to the Binaries folder.
if(UNIX)
	option(LINUX_LOCAL_DEV "Enable relocatable binary" OFF)
	if(LINUX_LOCAL_DEV)
		add_definitions('-DLINUX_LOCAL_DEV')
	endif(LINUX_LOCAL_DEV)
endif(UNIX)

# Dolphin requires threads.
# The Apple build may not need an explicit flag because one of the
# frameworks may already provide it.
# But for non-OSX systems, we will use the CMake Threads package.
IF(NOT APPLE)
  FIND_PACKAGE(Threads)
ENDIF(NOT APPLE)

if(NOT CMAKE_BUILD_TYPE)
	set(CMAKE_BUILD_TYPE "Release" CACHE STRING
		"Build type (Release/Debug/RelWithDebInfo/MinSizeRe)" FORCE)
endif(NOT CMAKE_BUILD_TYPE)


if(CMAKE_BUILD_TYPE STREQUAL Debug)
	add_definitions(-D_DEBUG -ggdb)
	set(wxWidgets_USE_DEBUG ON CACHE BOOL "Use wxWidgets Debugging")

	option(ENABLE_GPROF "Enable gprof profiling (must be using Debug build)" OFF)
	if(ENABLE_GPROF)
		add_definitions(-pg)
		set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -pg")
	endif()
endif(CMAKE_BUILD_TYPE STREQUAL Debug)

if(CMAKE_BUILD_TYPE STREQUAL Release AND NOT APPLE)
	add_definitions(-fomit-frame-pointer)
endif()

if(FASTLOG)
	add_definitions(-DDEBUGFAST)
endif()

if(GDBSTUB)
	add_definitions(-DUSE_GDBSTUB)
endif(GDBSTUB)

if(VTUNE)
	if(EXISTS "$ENV{VTUNE_AMPLIFIER_XE_2015_DIR}")
		set(VTUNE_DIR "$ENV{VTUNE_AMPLIFIER_XE_2015_DIR}")
	elseif(EXISTS "$ENV{VTUNE_AMPLIFIER_XE_2013_DIR}")
		set(VTUNE_DIR "$ENV{VTUNE_AMPLIFIER_XE_2013_DIR}")
	else()
		message(ERROR "Could find neither VTUNE_AMPLIFIER_XE_2015_DIR nor VTUNE_AMPLIFIER_XE_2013_DIR.")
	endif()
	add_definitions(-DUSE_VTUNE)
	include_directories("${VTUNE_DIR}/include")
	set(VTUNE_LIBRARIES
		"${VTUNE_DIR}/lib64/libjitprofiling.a"
		"${VTUNE_DIR}/lib64/libittnotify.a"
	)
endif(VTUNE)

if(ANDROID)
	message("Building for Android")
	if(NOT ENABLE_HEADLESS)
		add_definitions(-DANDROID)
	else()
		# Lie to cmake a bit. We are cross compiling to Android
		# but not as a shared library. We want an executable.
		set(ANDROID 0)
	endif()
	set(USE_X11 0)
	set(USE_UPNP 0)
	set(USE_EGL 1)
	set(DISABLE_WX 1)

	# We are cross compiling, search only the toolchain for libraries and includes
	SET(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
	SET(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
elseif(NOT APPLE)
	list(APPEND LIBS rt)
endif()

if(ENABLE_HEADLESS)
	message("Enabling Headless! Disabling GUI, force enabling EGL!")
	set(USE_X11 0)
	set(USE_EGL 1)
	set(DISABLE_WX 1)
	add_definitions(-DUSE_HEADLESS)
endif()

add_definitions(-D_FILE_OFFSET_BITS=64 -D_LARGEFILE_SOURCE)

########################################
# Dependency checking
#
# TODO: We should have options for dependencies included in the externals to
# override autodetection of system libraries and force the usage of the
# externals.
include(CheckLib)
include(CheckCXXSourceRuns)

include(FindOpenGL)
if (OPENGL_GL)
	include_directories(${OPENGL_INCLUDE_DIR})
endif()

include(FindALSA OPTIONAL)
if(ALSA_FOUND)
	add_definitions(-DHAVE_ALSA=1)
	message("ALSA found, enabling ALSA sound backend")
else()
	add_definitions(-DHAVE_ALSA=0)
	message("ALSA NOT found, disabling ALSA sound backend")
endif(ALSA_FOUND)

check_lib(AO ao ao QUIET)
if(AO_FOUND)
	add_definitions(-DHAVE_AO=1)
	message("ao found, enabling ao sound backend")
else()
	add_definitions(-DHAVE_AO=0)
	message("ao NOT found, disabling ao sound backend")
endif(AO_FOUND)

check_lib(BLUEZ bluez bluez QUIET)
if(BLUEZ_FOUND)
	add_definitions(-DHAVE_BLUEZ=1)
	message("bluez found, enabling bluetooth support")
else()
	add_definitions(-DHAVE_BLUEZ=0)
	message("bluez NOT found, disabling bluetooth support")
endif(BLUEZ_FOUND)

check_lib(PULSEAUDIO libpulse pulse QUIET)
if(PULSEAUDIO_FOUND)
	add_definitions(-DHAVE_PULSEAUDIO=1)
	message("PulseAudio found, enabling PulseAudio sound backend")
else()
	add_definitions(-DHAVE_PULSEAUDIO=0)
	message("PulseAudio NOT found, disabling PulseAudio sound backend")
endif(PULSEAUDIO_FOUND)

include(FindOpenAL OPTIONAL)
if(OPENAL_FOUND)
	add_definitions(-DHAVE_OPENAL=1)
	include_directories(${OPENAL_INCLUDE_DIR})
	message("OpenAL found, enabling OpenAL sound backend")
else()
	add_definitions(-DHAVE_OPENAL=0)
	message("OpenAL NOT found, disabling OpenAL sound backend")
endif(OPENAL_FOUND)

include(FindLLVM OPTIONAL)
if (LLVM_FOUND)
	add_definitions(-DHAS_LLVM=1)
	set(HAS_LLVM 1)

	include_directories(${LLVM_INCLUDE_DIRS})
	list(APPEND LIBS ${LLVM_LIBRARIES})

	message(STATUS "Found LLVM ${LLVM_PACKAGE_VERSION}")
endif()

set(USE_X11 0)

if(UNIX AND NOT APPLE AND NOT ANDROID AND NOT ENABLE_HEADLESS)
	include(FindX11)
	if(TRY_X11 AND X11_FOUND)
		set(USE_X11 1)
		add_definitions(-DHAVE_X11=1)
		include_directories(${X11_INCLUDE_DIR})
		message("X11 support enabled")
	else()
		set(USE_X11 0)
		SET(X11_FOUND "")
		message("X11 support disabled")
		add_definitions(-DHAVE_X11=0)
	endif(TRY_X11 AND X11_FOUND)

	if (NOT USE_X11)
		message(FATAL_ERROR "\n"
			"No suitable display platform found\n"
			"Requires x11 to run")
	endif()
endif()

if(USE_X11)
	check_lib(XRANDR xrandr Xrandr)
	if(XRANDR_FOUND)
		add_definitions(-DHAVE_XRANDR=1)
	else()
		add_definitions(-DHAVE_XRANDR=0)
	endif(XRANDR_FOUND)

	pkg_check_modules(XINPUT2 xi>=1.5.0)
	if(XINPUT2_FOUND)
		add_definitions(-DHAVE_X11_XINPUT2=1)
	else()
		add_definitions(-DHAVE_X11_XINPUT2=0)
	endif(XINPUT2_FOUND)
endif()
if(ENCODE_FRAMEDUMPS)
	check_libav()
	if(LIBAV_FOUND)
		LIST(APPEND LIBS ${LIBAV_LDFLAGS})
	endif()

endif()

set(CMAKE_REQUIRED_LIBRARIES portaudio)
CHECK_CXX_SOURCE_RUNS(
	"#include <portaudio.h>
	int main(int argc, char **argv)
	{ if(Pa_GetVersion() >= 1890) return 0; else return 1; }"
	PORTAUDIO)
unset(CMAKE_REQUIRED_LIBRARIES)
if(PORTAUDIO)
	message("PortAudio found, enabling mic support")
	add_definitions(-DHAVE_PORTAUDIO=1)
	set(PORTAUDIO_FOUND TRUE)
else()
	message("PortAudio not found, disabling mic support")
	add_definitions(-DHAVE_PORTAUDIO=0)
	set(PORTAUDIO_FOUND FALSE)
endif(PORTAUDIO)

if(OPROFILING)
	include(FindOProfile)
	if(OPROFILE_FOUND)
		message("OProfile found, enabling profiling support")
		add_definitions(-DUSE_OPROFILE=1)
		include_directories(${OPROFILE_INCLUDE_DIRS})
	else()
		message(FATAL_ERROR "OProfile not found. Can't build profiling support.")
	endif()
endif()

if(USE_EGL)
	message("EGL OpenGL interface enabled")
	add_definitions(-DUSE_EGL=1)
endif()

if(ENABLE_EVDEV)
	include(FindLibudev REQUIRED)
	include(FindLibevdev REQUIRED)
	if(LIBUDEV_FOUND AND LIBEVDEV_FOUND)
		message("libevdev/libudev found, enabling evdev controller backend")
		add_definitions(-DHAVE_LIBUDEV=1)
		add_definitions(-DHAVE_LIBEVDEV=1)
		include_directories(${LIBUDEV_INCLUDE_DIR} ${LIBEVDEV_INCLUDE_DIR})
	else()
		message(FATAL_ERROR "Couldn't find libevdev and/or libudev. Can't build evdev controller backend.\nDisable ENABLE_EVDEV if you wish to build without controller support")
	endif()
endif()

if(UNIX)
	message("Using named pipes as controller inputs")
	add_definitions(-DUSE_PIPES=1)
	message("Watching game memory for changes")
	add_definitions(-DUSE_MEMORYWATCHER=1)
endif()

if(ENABLE_ANALYTICS)
	message("Enabling analytics collection (subject to end-user opt-in)")
	add_definitions(-DUSE_ANALYTICS=1)
endif()

########################################
# Setup include directories (and make sure they are preferred over the Externals)
#
include_directories(Source/Core)
if(ANDROID)
	include_directories(Source/Android)
endif()

########################################
# Process externals and setup their include directories
#
# NOTES about adding Externals:
#   - add the include directory here
#   - make sure to tell cmake to link them statically or dynamically (most
#     should be linked statically)
#   - place the CMakeLists.txt in the first-level subdirectory, e.g.
#     Externals/zlib/CMakeLists.txt (that is: NOT in some Src/ subdirectory)
#
add_subdirectory(Externals/Bochs_disasm)
include_directories(Externals/Bochs_disasm)
add_subdirectory(Externals/lua-5.3.1)
include_directories(Externals/lua-5.3.1/src)

if(USE_SHARED_ENET)
	check_lib(ENET libenet enet enet/enet.h QUIET)
	include(CheckSymbolExists)
	if (ENET_FOUND)
		set(CMAKE_REQUIRED_INCLUDES ${ENET_INCLUDE_DIRS})
		# hack: LDFLAGS already contains -lenet but all flags but the first are
		# dropped; ugh, cmake
		set(CMAKE_REQUIRED_FLAGS ${ENET_LDFLAGS})
		set(CMAKE_REQUIRED_LIBRARIES ${ENET_LIBRARIES})
		CHECK_SYMBOL_EXISTS(enet_socket_get_address enet/enet.h ENET_HAVE_SGA)
		set(CMAKE_REQUIRED_INCLUDES)
		set(CMAKE_REQUIRED_FLAGS)
		set(CMAKE_REQUIRED_LIBRARIES)
		if (NOT ENET_HAVE_SGA)
			# enet is too old
			set(ENET_FOUND FALSE)
		endif()
	endif()
endif()
if (ENET_FOUND)
	message("Using shared enet")
else()
	message("Using static enet from Externals")
	include_directories(Externals/enet/include)
	add_subdirectory(Externals/enet)
endif()
LIST(APPEND LIBS enet)

if(NOT XXHASH_FOUND)
	message("Using static xxhash from Externals")
	add_subdirectory(Externals/xxhash)
	include_directories(Externals/xxhash)
endif()
LIST(APPEND LIBS xxhash)

# If zlib has already been found on a previous run of cmake don't check again
# as the check seems to take a long time.
if(NOT ZLIB_FOUND)
	include(FindZLIB OPTIONAL)
endif()
if(ZLIB_FOUND)
	set(ZLIB_FOUND 1 CACHE INTERNAL "")
	message("Using shared zlib")
	include_directories(${ZLIB_INCLUDE_DIRS})
else(ZLIB_FOUND)
	message("Shared zlib not found, falling back to the static library")
	add_subdirectory(Externals/zlib)
	include_directories(Externals/zlib)
endif(ZLIB_FOUND)

if(NOT APPLE)
	check_lib(LZO "(no .pc for lzo2)" lzo2 lzo/lzo1x.h QUIET)
endif()
if(LZO_FOUND)
	message("Using shared lzo")
else()
	message("Using static lzo from Externals")
	add_subdirectory(Externals/LZO)
	include_directories(Externals/LZO)
	set(LZO lzo2)
endif()
list(APPEND LIBS ${LZO})

if(NOT APPLE)
	check_lib(PNG libpng png png.h QUIET)
endif()
if (PNG_FOUND)
	message("Using shared libpng")
else()
	message("Using static libpng from Externals")
	add_subdirectory(Externals/libpng)
	include_directories(Externals/libpng)
	set(PNG png)
endif()

if(OPENAL_FOUND)
	if(NOT APPLE)
		check_lib(SOUNDTOUCH soundtouch SoundTouch soundtouch/SoundTouch.h QUIET)
	endif()
	if (SOUNDTOUCH_FOUND)
		message("Using shared soundtouch")
	else()
		message("Using static soundtouch from Externals")
		add_subdirectory(Externals/soundtouch)
		include_directories(Externals)
	endif()
endif()

if(ENABLE_SDL)
	include(FindSDL2 OPTIONAL)
	if(SDL2_FOUND)
		message("Using shared SDL2")
		add_definitions(-DHAVE_SDL=1)
		include_directories(${SDL2_INCLUDE_DIR})
	else(SDL2_FOUND)
		# SDL2 not found, try SDL
		include(FindSDL OPTIONAL)
		if(SDL_FOUND)
			message("Using shared SDL")
			add_definitions(-DHAVE_SDL=1)
			include_directories(${SDL_INCLUDE_DIR})
		else(SDL_FOUND)
			message("SDL NOT found, disabling SDL input")
		endif(SDL_FOUND)
	endif(SDL2_FOUND)
endif()

include(FindLibUSB OPTIONAL)
if(LIBUSB_FOUND)
	message("Using shared LibUSB")
	add_definitions(-D__LIBUSB__)
	include_directories(${LIBUSB_INCLUDE_DIR})
endif(LIBUSB_FOUND)

set(SFML_REQD_VERSION 2.1)
if(NOT APPLE)
	find_package(SFML ${SFML_REQD_VERSION} COMPONENTS network system)
endif()
if(SFML_FOUND)
	message("Using shared SFML")
else()
	message("Using static SFML ${SFML_REQD_VERSION} from Externals")
	add_definitions(-DSFML_STATIC)
	add_subdirectory(Externals/SFML)
	include_directories(BEFORE Externals/SFML/include)
endif()

if(USE_UPNP)
	if(NOT APPLE)
		include(FindMiniupnpc)
	endif()
	if(MINIUPNPC_FOUND AND MINIUPNPC_API_VERSION GREATER 8)
		message("Using shared miniupnpc")
	else()
		message("Using static miniupnpc from Externals")
		add_subdirectory(Externals/miniupnpc)
		set(MINIUPNPC_INCLUDE_DIRS Externals/miniupnpc/src)
		set(MINIUPNPC_LIBRARIES miniupnpc)
	endif()
	add_definitions(-DUSE_UPNP)
	include_directories(${MINIUPNPC_INCLUDE_DIRS})
	list(APPEND LIBS ${MINIUPNPC_LIBRARIES})
endif()

if(NOT APPLE)
	include(FindMbedTLS)
endif()
if(MBEDTLS_FOUND)
	message("Using shared mbed TLS")
	include_directories(${MBEDTLS_INCLUDE_DIRS})
else()
	message("Using static mbed TLS from Externals")
	set(MBEDTLS_LIBRARIES mbedtls mbedcrypto mbedx509)
	add_subdirectory(Externals/mbedtls/)
	include_directories(Externals/mbedtls/include)
endif()

include(FindCURL OPTIONAL)
if(CURL_FOUND)
	message("Using shared libcurl")
	include_directories(${CURL_INCLUDE_DIRS})
else()
	message("Using static libcurl from Externals")
	add_subdirectory(Externals/curl)
	set(CURL_LIBRARIES curl)
	include_directories(BEFORE Externals/curl/include)
endif()

if(NOT APPLE)
	check_lib(SOIL "(no .pc for SOIL)" SOIL SOIL/SOIL.h QUIET)
endif()
if(SOIL_FOUND)
	message("Using shared SOIL")
else()
	message("Using static SOIL from Externals")
	add_subdirectory(Externals/SOIL)
	include_directories(Externals/SOIL)
endif()

find_library(ICONV_LIBRARIES NAMES iconv libiconv libiconv-2 c)
find_path(ICONV_INCLUDE_DIR NAMES iconv.h)

if (ICONV_LIBRARIES AND ICONV_INCLUDE_DIR)
	mark_as_advanced(ICONV_INCLUDE_DIR ICONV_LIBRARIES)
else()
	message("Using static iconv from Externals")
	include_directories(Externals/libiconv-1.14/include)
	add_subdirectory(Externals/libiconv-1.14)
	set(ICONV_LIBRARIES iconv)
endif()
list(APPEND LIBS ${ICONV_LIBRARIES})

find_library(OPENSLES_LIBRARIES NAMES OpenSLES)
find_path(OPENSLES_INCLUDE_DIR NAMES SLES/OpenSLES.h)

if (OPENSLES_LIBRARIES AND OPENSLES_INCLUDE_DIR)
	set(OPENSLES_FOUND 1)
	add_definitions(-DHAVE_OPENSLES=1)
	include_directories(${OPENSLES_INCLUDE_DIR})
	message("OpenSLES found, enabling OpenSLES sound backend")
endif()

if(NOT DISABLE_WX)
	include(FindwxWidgets OPTIONAL)
	FIND_PACKAGE(wxWidgets COMPONENTS core aui adv)

	if(wxWidgets_FOUND)
		EXECUTE_PROCESS(WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
			COMMAND sh "${wxWidgets_CONFIG_EXECUTABLE}"
			${wxWidgets_CONFIG_OPTIONS} --version
			OUTPUT_VARIABLE wxWidgets_VERSION
			OUTPUT_STRIP_TRAILING_WHITESPACE
			ERROR_QUIET
			)
		message("Found wxWidgets version ${wxWidgets_VERSION}")
		set(wxMIN_VERSION "3.0.1")
		if(${wxWidgets_VERSION} VERSION_LESS ${wxMIN_VERSION})
			message("At least ${wxMIN_VERSION} is required; ignoring found version")
			unset(wxWidgets_FOUND)
		endif()
	endif(wxWidgets_FOUND)

	if(UNIX AND NOT APPLE)
		# There is a bug in the FindGTK module in cmake version 2.8.2 that
		# does not find gdk-pixbuf-2.0.  On the other hand some 2.8.3
		# users have complained that pkg-config does not find
		# gdk-pixbuf-2.0.  On yet another hand, cmake version 2.8.3 in
		# Ubuntu Natty does not find the glib libraries correctly.
		# Ugly!!!
		execute_process(COMMAND lsb_release -c -s
			OUTPUT_VARIABLE DIST_NAME
			ERROR_QUIET OUTPUT_STRIP_TRAILING_WHITESPACE)
		if(${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION}.${CMAKE_PATCH_VERSION}
				VERSION_EQUAL 2.8.2 OR "${DIST_NAME}" STREQUAL "natty")
			check_lib(GTK2 gtk+-2.0 gtk+-2.0 gtk.h REQUIRED)
		else()
			include(FindGTK2)
			if(GTK2_FOUND)
				include_directories(${GTK2_INCLUDE_DIRS})
				list(APPEND LIBS ${GTK2_LIBRARIES})
			else()
				message(FATAL_ERROR "GTK is required to build the WX UI. Please install the GTK development libraries.")
			endif()
		endif()
	endif()

	if(wxWidgets_FOUND)
		include(${wxWidgets_USE_FILE})
		message("wxWidgets found, enabling GUI build")
	else(wxWidgets_FOUND)
		message("Using static wxWidgets from Externals")

		# These definitions and includes are used when building dolphin against wx,
		# not when building wx itself (see wxw3 CMakeLists.txt for that)
		if(APPLE)
			add_definitions(-D__WXOSX_COCOA__)
		elseif(${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
			add_definitions(-D__WXGTK__)

			# Check for required libs
			check_lib(GTHREAD2 gthread-2.0 gthread-2.0 glib/gthread.h REQUIRED)
			check_lib(PANGOCAIRO pangocairo pangocairo pango/pangocairo.h REQUIRED)
		elseif(WIN32)
			add_definitions(-D__WXMSW__)
		else()
			message(FATAL_ERROR "wxWidgets in Externals is not compatible with your platform")
		endif()

		include_directories(SYSTEM
			Externals/wxWidgets3
			Externals/wxWidgets3/include)
		add_subdirectory(Externals/wxWidgets3)
		set(wxWidgets_FOUND TRUE)
		set(wxWidgets_LIBRARIES "wx")
	endif(wxWidgets_FOUND)
	add_definitions(-DHAVE_WX=1)
endif(NOT DISABLE_WX)

if(${CMAKE_SYSTEM_NAME} MATCHES "FreeBSD|NetBSD")
	set(LIBS ${LIBS} usbhid)
endif()

if(APPLE)
	# Link against OS X system frameworks.
	list(APPEND LIBS
		${APPKIT_LIBRARY}
		${AU_LIBRARY}
		${COREAUDIO_LIBRARY}
		${COREFUND_LIBRARY}
		${CORESERV_LIBRARY}
		${IOK_LIBRARY}
		${FORCEFEEDBACK}
		)
endif()

########################################
# Pre-build events: Define configuration variables and write SCM info header
#
if(DOLPHIN_WC_BRANCH STREQUAL "master" OR DOLPHIN_WC_BRANCH STREQUAL "stable")
	set(DOLPHIN_WC_IS_STABLE "1")
else()
	set(DOLPHIN_WC_IS_STABLE "0")
endif()

file(WRITE ${PROJECT_BINARY_DIR}/Source/Core/Common/scmrev.h
	"#define SCM_REV_STR \"" ${DOLPHIN_WC_REVISION} "\"\n"
	"#define SCM_DESC_STR \"" ${DOLPHIN_WC_DESCRIBE} "\"\n"
	"#define SCM_BRANCH_STR \"" ${DOLPHIN_WC_BRANCH} "\"\n"
	"#define SCM_IS_MASTER " ${DOLPHIN_WC_IS_STABLE} "\n"
	"#define SCM_DISTRIBUTOR_STR \"" ${DISTRIBUTOR} "\"\n"
	)
include_directories("${PROJECT_BINARY_DIR}/Source/Core/Common")

########################################
# Unit testing.
#
include(FindGTest)
if(GTEST_FOUND AND USE_SHARED_GTEST)
	message("Using shared gtest")
	include_directories(${GTEST_INCLUDE_DIRS})
else()
	message("Using static gtest from Externals")
	include_directories(Externals/gtest/include)
	add_subdirectory(Externals/gtest)
endif()

enable_testing()
add_custom_target(unittests)
add_custom_command(TARGET unittests POST_BUILD COMMAND ${CMAKE_CTEST_COMMAND})


########################################
# Start compiling our code
#
add_definitions(-std=c++1y)

# These aren't actually needed for C11/C++11
# but some dependencies require them (LLVM, libav).
add_definitions(-D__STDC_LIMIT_MACROS -D__STDC_CONSTANT_MACROS)

# Do this at the last minute because try_compile ignores linker flags.  Yay...
if(APPLE)
	# Some of our code contains Objective C constructs.
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -x objective-c -stdlib=libc++")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -x objective-c++ -stdlib=libc++")
	# Avoid mistaking an object file for a source file on the link command line.
	set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -x none")
endif()

add_subdirectory(Source)


########################################
# Install shared data files
#
if(NOT APPLE)
	install(DIRECTORY Data/Sys/ DESTINATION ${datadir}/sys PATTERN)
  install(DIRECTORY Data/Sys/Scripts/ DESTINATION ${datadir}/sys/Scripts PATTERN)
	install(DIRECTORY Data/Sys/InfoDisplay/ DESTINATION ${datadir}/sys/InfoDisplay PATTERN)
endif()
if(NOT ${CMAKE_SYSTEM_NAME} MATCHES "Linux|FreeBSD|Darwin")
	install(FILES Data/license.txt DESTINATION ${datadir})
endif()
if(${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
	# Install the application icon and menu item
	install(FILES Data/dolphin-emu.svg
		DESTINATION ${CMAKE_INSTALL_PREFIX}/share/icons/hicolor/scalable/apps)
	install(FILES Data/dolphin-emu.png
		DESTINATION ${CMAKE_INSTALL_PREFIX}/share/icons/hicolor/48x48/apps)
	install(FILES Data/dolphin-emu.desktop
		DESTINATION ${CMAKE_INSTALL_PREFIX}/share/applications)
	# Install manpages
	install(FILES Data/dolphin-emu.6
		DESTINATION ${mandir}/man6)
endif()

# packaging information
set(CPACK_PACKAGE_NAME "dolphin-emu")
set(CPACK_PACKAGE_VENDOR "Dolphin Team")
set(CPACK_PACKAGE_VERSION_MAJOR ${DOLPHIN_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${DOLPHIN_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${DOLPHIN_VERSION_PATCH})
set(CPACK_PACKAGE_DESCRIPTION_FILE ${PROJECT_SOURCE_DIR}/Data/cpack_package_description.txt)
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "A GameCube, Wii and Triforce emulator")

set(CPACK_RPM_PACKAGE_GROUP System/Emulators/Other)
set(CPACK_RPM_PACKAGE_LICENSE GPL-2.0)
# TODO: CPACK_RESOURCE_FILE_README
# TODO: CPACK_RESOURCE_FILE_WELCOME
# TODO: CPACK_PACKAGE_ICON
# TODO: CPACK_NSIS_*
# TODO: Use CPack components for DSPSpy, etc => cpack_add_component

set(CPACK_SET_DESTDIR ON)
set(CPACK_SOURCE_GENERATOR "TGZ;TBZ2;ZIP")
set(CPACK_SOURCE_IGNORE_FILES  "\\\\.#;/#;.*~;\\\\.swp;/\\\\.git")
list(APPEND CPACK_SOURCE_IGNORE_FILES "${CMAKE_BINARY_DIR}")

# CPack must be included after the CPACK_* variables are set in order for those
# variables to take effect.
Include(CPack)

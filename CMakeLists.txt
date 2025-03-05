cmake_minimum_required (VERSION 3.14)
project(
	printf
	LANGUAGES C
	DESCRIPTION "Self-contained C implementation of printf, vprintf, sprintf and related functions"
	HOMEPAGE_URL https://github.com/eyalroz/printf
	VERSION 6.0.0
)

#########################
# Project Build Options #
#########################

option(BUILD_TESTS          "Build test programs for the library" ON)
option(BUILD_SHARED_LIBS    "Build the library as a shared (dynamically-linked) object rather than a static one " OFF)

########################
# Type Size Validation #
########################

include(CheckTypeSize)

# Checks related to the 'j', 'z' and 't' size modifiers

check_type_size( "long"        SIZEOF_LONG       )
check_type_size( "long long"   SIZEOF_LONG_LONG  )

set(ACCEPTABLE_JZT_TYPE_SIZES ${SIZEOF_LONG} ${SIZEOF_LONG_LONG})

function(validate_type_size type_name)
	check_type_size(${type_name} TYPE_SIZE)
	if (NOT ${TYPE_SIZE} IN_LIST ACCEPTABLE_JZT_TYPE_SIZES)
		message(FATAL_ERROR "sizeof(${type_name}) is  ${TYPE_SIZE}, which is neither sizeof(long) (${SIZEOF_LONG}) nor sizeof(long long) (${SIZEOF_LONG_LONG}). Please contact the library maintainers with your platform details.")
	endif()
endfunction()
validate_type_size("intmax_t")
validate_type_size("size_t")
validate_type_size("ptrdiff_t")

######################
# Configuration File #
######################
add_subdirectory(config)

##################
# Library Target #
##################
add_library(printf)
add_library("printf::printf" ALIAS printf)

target_sources(printf PRIVATE src/printf/printf.c "${CONFIG_HEADER_DIR}/printf_config.h" src/printf/printf.h)
target_compile_definitions(printf PRIVATE PRINTF_INCLUDE_CONFIG_H)
target_include_directories(printf PRIVATE "$<BUILD_INTERFACE:${CONFIG_HEADER_DIR}>")

if (BUILD_SHARED_LIBS)
	set_property(TARGET printf PROPERTY VERSION ${PROJECT_VERSION})
	set_property(TARGET printf PROPERTY SOVERSION ${PROJECT_VERSION})
endif()

set_property(TARGET printf PROPERTY C_STANDARD 99)
set_property(TARGET printf PROPERTY C_STANDARD_REQUIRED ON)
set_property(TARGET printf PROPERTY C_EXTENSIONS OFF)

target_include_directories(
	printf
	PRIVATE
	src
	PUBLIC
	"$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>"
	"$<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/src/>"
)

set_target_properties(printf PROPERTIES
	LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib
	ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)

if (CMAKE_C_COMPILER_ID STREQUAL "MSVC")
	target_compile_options(printf PRIVATE /W4)
elseif (CMAKE_C_COMPILER_ID STREQUAL "GNU" OR
        CMAKE_C_COMPILER_ID STREQUAL "Clang")
	target_compile_options(printf PRIVATE -Wall -Wextra -pedantic -Wconversion)
	if (ALIAS_STANDARD_FUNCTION_NAMES)
		# This is important for preventing our aliased implementation
		# from being replaced, e.g. printf("%c", 'a') by putchar('a');
		# clang and GCC apparently do this as an optimization
		target_compile_options(printf PUBLIC -fno-builtin-printf)
	endif()
endif()

###########
# Testing #
###########

include(cmake/test/catch2.cmake)

if (BUILD_TESTS)
	enable_testing()
	add_subdirectory(test)
endif()

#######################
# Supporting Commands #
#######################

if (UNIX)
	add_custom_target(printf-sizes
		COMMAND size -A -t $<TARGET_FILE:printf>  > printf_sizes.txt
		DEPENDS printf
		BYPRODUCTS printf_sizes.txt
		COMMENT Prints the sizes of the different sections of the ELF file: text, dat, vss etc.)

	add_custom_target(printf-symbols
		COMMAND nm --numeric-sort --print-size "$<TARGET_FILE:printf>" > printf_symbols.txt
		COMMAND bash -c "nm --numeric-sort --print-size $<TARGET_FILE:printf> | c++filt > printf_cpp_symbols.txt"
		VERBATIM
		DEPENDS printf
		BYPRODUCTS printf_symbols.txt printf_cpp_symbols.txt
		COMMENT Produces lists of the symbols, and C++demangled symbols, inside the library)

	add_custom_target(printf-lst
		COMMAND objdump --disassemble --line-numbers -S "$<TARGET_FILE:printf>" > printf.list
		DEPENDS printf
		BYPRODUCTS printf.lst
		COMMENT Dissassembles the compiled library into an .lst file)
endif()

################
# Installation #
################

include(GNUInstallDirs)

# Note: No need for a config.cmake file for setting dependencies - as there
# are no dependencies; this library is self-contained

install(
	TARGETS printf
	EXPORT printf_export
	RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
	ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
	LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
	INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)

install(
	FILES "src/printf/printf.h"
	DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/printf"
)

export(
	EXPORT printf_export
	NAMESPACE "printf::"
	FILE "${PROJECT_BINARY_DIR}/printf-targets.cmake"
)

install(
	EXPORT printf_export
	FILE "printf-targets.cmake"
	DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/printf"
	NAMESPACE "printf::"
)

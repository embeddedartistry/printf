###########
# Options #
###########

# Boolean options which go into config.h

option(SUPPORT_DECIMAL_SPECIFIERS             "Support decimal notation floating-point conversion specifiers (%f,%F)" ON)
option(SUPPORT_EXPONENTIAL_SPECIFIERS         "Support exponential floating point format conversion specifiers (%e,%E,%g,%G)" ON)
option(SUPPORT_MSVC_STYLE_INTEGER_SPECIFIERS  "Support the I + bit size integer specifiers (%I8, %I16, %I32, %I64) as in Microsoft Visual C++" ON)
option(SUPPORT_WRITEBACK_SPECIFIER            "Support the length write-back specifier (%n)" ON)
option(SUPPORT_LONG_LONG                      "Support long long integral types (allows for the ll length modifier and affects %p)" ON)
option(USE_DOUBLE_INTERNALLY                  "Use the C `double` type - typically 64-bit in size - for internal floating-point arithmetic " ON)
option(CHECK_FOR_NUL_IN_FORMAT_SPECIFIER      "Be defensive in the undefined-behavior case of a format specifier not ending before the string ends" ON)

set(ALIASING_MODES NONE HARD SOFT)
set(ALIAS_STANDARD_FUNCTION_NAMES NONE CACHE STRING "Alias the standard library function names (printf, sprintf etc.) to the library's functions - concretely, via a macro, or not at all")
set_property(CACHE ALIAS_STANDARD_FUNCTION_NAMES PROPERTY STRINGS ${ALIASING_MODES})

if (NOT ${ALIAS_STANDARD_FUNCTION_NAMES} STREQUAL NONE)
	set("ALIAS_STANDARD_FUNCTION_NAMES_${ALIAS_STANDARD_FUNCTION_NAMES}" 1)
endif()

foreach(opt
	SUPPORT_DECIMAL_SPECIFIERS
	SUPPORT_EXPONENTIAL_SPECIFIERS
	SUPPORT_MSVC_STYLE_INTEGER_SPECIFIERS
	SUPPORT_WRITEBACK_SPECIFIER
	SUPPORT_LONG_LONG
	USE_DOUBLE_INTERNALLY
	ALIAS_STANDARD_FUNCTION_NAMES_SOFT
	ALIAS_STANDARD_FUNCTION_NAMES_HARD
	CHECK_FOR_NUL_IN_FORMAT_SPECIFIER
        )
	if (${${opt}})
		set("PRINTF_${opt}" 1)
	else()
		set("PRINTF_${opt}" 0)
	endif()
endforeach()

# Numeric defines which go into printf_config.h

set(PRINTF_INTEGER_BUFFER_SIZE     "32" CACHE STRING "Integer to string conversion buffer size")
set(PRINTF_DECIMAL_BUFFER_SIZE     "32" CACHE STRING "Floating-point to decimal conversion buffer size")
set(DEFAULT_FLOAT_PRECISION         "6" CACHE STRING "Default precision when printing floating-point values")
set(MAX_INTEGRAL_DIGITS_FOR_DECIMAL "9" CACHE STRING "Maximum number of integral-part digits of a floating-point value for which printing with %f uses decimal (non-exponential) notation")
set(LOG10_TAYLOR_TERMS              "4" CACHE STRING "The number of terms in a Taylor series expansion of log_10(x) to use for approximation")

###################
# Generate Header #
###################

set(CONFIG_HEADER_DIR "${CMAKE_CURRENT_BINARY_DIR}/include" CACHE INTERNAL "")
configure_file("printf_config.h.in" "${CONFIG_HEADER_DIR}/printf_config.h" @ONLY)

#########################
# Package config header #
#########################

# For now, these go in CMAKE_BINARY_DIR to preserve the previous
# output structure. They *could* go into config/ with CMAKE_CURRENT_BINARY_DIR
# though maybe that doesn't really work.

include(CMakePackageConfigHelpers)

configure_package_config_file(
	"printf-config.cmake.in"
	"${CMAKE_BINARY_DIR}/printf-config.cmake"
	INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/printf"
)

write_basic_package_version_file(
	"${CMAKE_BINARY_DIR}/printf-config-version.cmake"
	VERSION ${PROJECT_VERSION}
	COMPATIBILITY SameMinorVersion
)

install(
	FILES
		"${CMAKE_BINARY_DIR}/printf-config-version.cmake"
		"${CMAKE_BINARY_DIR}/printf-config.cmake"
	DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/printf"
)

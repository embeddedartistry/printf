cmake_minimum_required(VERSION 3.14)

enable_language(CXX)

#############################
# Primary catch2 Test Suite #
#############################

if (NOT ALIAS_STANDARD_FUNCTION_NAMES STREQUAL SOFT)
	add_executable(test_suite)
	target_sources(test_suite PRIVATE test_suite.cpp)
	target_link_libraries(test_suite PRIVATE catch2_dep)
	set_target_properties(
		test_suite
		PROPERTIES
		CXX_STANDARD 14
		CXX_STANDARD_REQUIRED YES
		CXX_EXTENSIONS NO
	)

	if (TEST_WITH_NON_STANDARD_FORMAT_STRINGS)
		target_compile_definitions(test_suite PRIVATE TEST_WITH_NON_STANDARD_FORMAT_STRINGS)
	endif()

	# These two lines are necessary, since the test suite does not actually use the
	# compiled library - it includes the library's source .c file; and that means we
	# need to include the generated config.h file.
	target_compile_definitions(test_suite PRIVATE PRINTF_INCLUDE_CONFIG_H)
	target_include_directories(
		test_suite
		PRIVATE
		"${CONFIG_HEADER_DIR}"
		"$<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/../src/>"
	)
endif()

option(TEST_WITH_NON_STANDARD_FORMAT_STRINGS "Include tests using non-standard-compliant format strings?" ON)
# ... don't worry, we'll suppress the compiler warnings for those.

#########################
# Comparison Test Suite #
#########################

add_executable(autotest autotest.cpp)
set_target_properties(
	autotest
	PROPERTIES
	CXX_STANDARD 14
	CXX_STANDARD_REQUIRED YES
	CXX_EXTENSIONS NO
)
target_link_libraries(autotest PRIVATE printf)
target_include_directories(autotest PRIVATE "$<BUILD_INTERFACE:${CONFIG_HEADER_DIR}>")

#######################
# Aliasing Test Suite #
#######################

add_executable(aliasing "aliasing.c")
set_target_properties(
	${tgt}
	PROPERTIES
	C_STANDARD 11
	C_STANDARD_REQUIRED YES
	C_EXTENSIONS NO
)

if (${CMAKE_C_COMPILER_ID} STREQUAL "GNU" OR ${CMAKE_C_COMPILER_ID} STREQUAL "Clang")
    target_compile_options(aliasing PRIVATE -fno-builtin-printf)
endif()

target_link_libraries(aliasing PRIVATE printf)
target_include_directories(aliasing PRIVATE "$<BUILD_INTERFACE:${CONFIG_HEADER_DIR}>")

###################
# Common Settings #
###################

list(APPEND test_targets aliasing test_suite autotest)

foreach(tgt ${test_targets})
	get_target_property(tgt_lang ${tgt} LANGUAGE)
	if (tgt_lang EQUAL "CXX")
		set(tgt_compiler_id ${CMAKE_CXX_COMPILER_ID})
	else() # it's C
		set(tgt_compiler_id ${CMAKE_C_COMPILER_ID})
	endif()

	if (tgt_compiler_id STREQUAL "MSVC")
		target_compile_options(${tgt} PRIVATE /W4)
	elseif (tgt_compiler_id STREQUAL "GNU" OR
			tgt_compiler_id STREQUAL "Clang")
		# lots of warnings and all warnings as errors
		target_compile_options(${tgt} PRIVATE
			-g
			-Wall
			-Wextra
			-pedantic
			-Wundef
			-Wsign-conversion
			-Wuninitialized
			-Wshadow
			-Wunreachable-code
			-Wswitch-default
			-Wswitch
			-Wcast-align
			-Wmissing-include-dirs
			-Winit-self
			-Wdouble-promotion
			-gdwarf-2
			-fno-exceptions
			-ffunction-sections
			-fdata-sections
			-fverbose-asm
			-Wunused-parameter
		)
		target_compile_options(${tgt} PRIVATE
		  "$<$<COMPILE_LANGUAGE:C>:-Wstrict-prototypes>"
		)			
		if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
			target_compile_options(${tgt} PRIVATE -ffat-lto-objects)
		endif()
	endif()

endforeach()

#####################
# Test Registration #
#####################

register_catch2_test(${PROJECT_NAME}.test_suite test_suite)

add_test(
	NAME "${PROJECT_NAME}.aliasing"
	COMMAND "aliasing"
)

# TODO - run autotest?
# Old status: Not running autotest by default - it's randomized after all.

cmake_minimum_required(VERSION 3.10)

project(jq)

list(APPEND DEFS "PACKAGE_NAME=\"jq\"")
list(APPEND DEFS "PACKAGE_TARNAME=\"jq\"")
list(APPEND DEFS "PACKAGE_VERSION=\"1.6\"")
list(APPEND DEFS "PACKAGE_STRING=\"jq\ 1.6\"")
list(APPEND DEFS "PACKAGE_BUGREPORT=\"https://github.com/stedolan/jq/issues\"")
list(APPEND DEFS "PACKAGE_URL=\"https://stedolan.github.io/jq\"")
list(APPEND DEFS "PACKAGE=\"jq\"")
list(APPEND DEFS "VERSION=\"1.6\"")

include(CheckIncludeFile)
foreach(HEADER_FILE sys/types.h sys/stat.h stdlib.h string.h memory.h strings.h inttypes.h stdint.h unistd.h alloca.h dlfcn.h)
	string(TOUPPER ${HEADER_FILE} HEADER_UPPER)
	string(REPLACE "/" "_" HEADER_UPPER ${HEADER_UPPER})
	string(REPLACE "." "_" HEADER_UPPER ${HEADER_UPPER})
	check_include_file(${HEADER_FILE} HAVE_${HEADER_UPPER})
	if(HAVE_${HEADER_UPPER})
		list(APPEND HAVE_FUNCS HAVE_${HEADER_UPPER})
	endif()
endforeach()

include(CheckSymbolExists)
macro(cmake_find_func func inc)
	string(TOUPPER ${func} FUNC_UPPER)
	check_symbol_exists(${func} ${inc} HAVE_${FUNC_UPPER})
	if(HAVE_${FUNC_UPPER})
		list(APPEND HAVE_FUNCS HAVE_${FUNC_UPPER})
	endif()
endmacro()
macro(cmake_find_func_def func inc def)
	set(CMAKE_REQUIRED_DEFINITIONS ${def})
	cmake_find_func(${func} ${inc})
	unset(CMAKE_REQUIRED_DEFINITIONS)
endmacro()
macro(cmake_find_math_func func)
	if(NOT MSVC)
		set(CMAKE_REQUIRED_LIBRARIES m)
	endif()
	cmake_find_func(${func} math.h)
	if(NOT MSVC)
		unset(CMAKE_REQUIRED_LIBRARIES)
	endif()
endmacro()
macro(cmake_find_math_func_def func def)
	set(CMAKE_REQUIRED_DEFINITIONS ${def})
	cmake_find_math_func(${func})
	unset(CMAKE_REQUIRED_DEFINITIONS)
endmacro()
cmake_find_func(alloca alloca.h)
cmake_find_func(isatty unistd.h)
cmake_find_func(_isatty io.h)
cmake_find_func_def(strptime time.h -D_XOPEN_SOURCE)
cmake_find_func(strftime time.h)
cmake_find_func(timegm time.h)
cmake_find_func(gmtime_r time.h)
cmake_find_func(gmtime time.h)
cmake_find_func(localtime_r time.h)
cmake_find_func(localtime time.h)
cmake_find_func(gettimeofday sys/time.h)
cmake_find_func_def(memmem string.h -D_GNU_SOURCE)
cmake_find_func_def(mkstemp stdlib.h -D_XOPEN_SOURCE=500)

include(CheckCSourceCompiles)
macro(cmake_check_member type member header var)
	check_c_source_compiles("
#include <${header}>
void main() {
	(void)((struct ${type} *)0)->${member};
}
" ${var})
	if(${var})
		list(APPEND HAVE_FUNCS ${var})
	endif()
endmacro()
cmake_check_member(tm tm_gmtoff time.h HAVE_TM_TM_GMT_OFF)
cmake_check_member(tm __tm_gmtoff time.h HAVE_TM___TM_GMT_OFF)

# Math functions
foreach(MATH_FUNC acos acosh asin asinh atan2 atan atanh cbrt ceil copysign cos cosh drem erf erfc exp2 exp expm1 fabs fdim floor fma fmax fmin fmod frexp gamma hypot j0 j1 jn ldexp lgamma log10 log1p log2 log logb modf lgamma_r nearbyint nextafter nexttoward pow10 pow remainder rint round scalb scalbln significand sin sinh sqrt tan tanh tgamma trunc y0 y1 yn)
	cmake_find_math_func(${MATH_FUNC})
endforeach()
cmake_find_math_func_def(exp10 -D_GNU_SOURCE)

# Thread local storage
check_c_source_compiles("
static __thread int a;
int main () { a = 12; return a; }
" HAVE___THREAD)
if(HAVE___THREAD)
	list(APPEND HAVE_FUNCS HAVE___THREAD)
endif()

# Endianness
include(TestBigEndian)
TEST_BIG_ENDIAN(IS_BIG_ENDIAN)
if(IS_BIG_ENDIAN)
	set(IEEE_MC68k 1)
	list(APPEND HAVE_FUNCS IEEE_MC68k)
else()
	set(IEEE_8087 1)
	list(APPEND HAVE_FUNCS IEEE_8087)
endif()

# Oniguruma
find_library(ONIG_LIBRARY onig)
if(ONIG_LIBRARY)
	message(STATUS libonig: ${ONIG_LIBRARY})
	set(HAVE_LIBONIG 1)
	list(APPEND HAVE_FUNCS HAVE_LIBONIG)
endif()

add_custom_command(OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/src/version.h"
	COMMAND "${CMAKE_COMMAND}" -E make_directory "${CMAKE_CURRENT_BINARY_DIR}/src"
	COMMAND "${CMAKE_COMMAND}" "-DFILE=${CMAKE_CURRENT_BINARY_DIR}/src/version.h" -P "${CMAKE_CURRENT_LIST_DIR}/jqversion.cmake"
)
add_custom_command(OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/src/builtin.inc" MAIN_DEPENDENCY src/builtin.jq
	COMMAND "${CMAKE_COMMAND}" -E make_directory "${CMAKE_CURRENT_BINARY_DIR}/src"
	COMMAND "${CMAKE_COMMAND}" "-DIN_FILE=${CMAKE_CURRENT_LIST_DIR}/src/builtin.jq" "-DOUT_FILE=${CMAKE_CURRENT_BINARY_DIR}/src/builtin.inc" -P "${CMAKE_CURRENT_LIST_DIR}/tocstring.cmake"
)
add_library(libjq
	src/builtin.c
	src/bytecode.c
	src/compile.c
	src/execute.c
	src/jq_test.c
	src/jv.c
	src/jv_alloc.c
	src/jv_aux.c
	src/jv_dtoa.c
	src/jv_file.c
	src/jv_parse.c
	src/jv_print.c
	src/jv_unicode.c
	src/lexer.c
	src/linker.c
	src/locfile.c
	src/parser.c
	src/util.c

	src/builtin.h
	src/bytecode.h
	src/compile.h
	src/exec_stack.h
	src/jq_parser.h
	src/jv_alloc.h
	src/jv_dtoa.h
	src/jv_unicode.h
	src/jv_utf8_tables.h
	src/lexer.l
	src/libm.h
	src/linker.h
	src/locfile.h
	src/opcode_list.h
	src/parser.y
	src/util.h
	${CMAKE_CURRENT_BINARY_DIR}/src/version.h
	${CMAKE_CURRENT_BINARY_DIR}/src/builtin.inc
)
target_compile_definitions(libjq PRIVATE ${DEFS})
foreach(HAVE_FUNC ${HAVE_FUNCS})
	target_compile_definitions(libjq PRIVATE ${HAVE_FUNC}=${${HAVE_FUNC}})
endforeach()
target_include_directories(libjq PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
if(MSVC)
	# MSVC 2019 C/C++ conformant preprocessor - https://devblogs.microsoft.com/cppblog/announcing-full-support-for-a-c-c-conformant-preprocessor-in-msvc/
	target_compile_options(libjq PRIVATE /Zc:preprocessor)
else()
	target_compile_options(libjq PRIVATE -Wextra -Wall -Wno-missing-field-initializers -Wno-unused-parameter -Wno-unused-function)
endif()
if(ONIG_LIBRARY)
	target_link_libraries(libjq PRIVATE ${ONIG_LIBRARY})
endif()
if(NOT MSVC)
	target_link_libraries(libjq PRIVATE m)
endif()

add_executable(jq src/main.c src/version.h)
target_compile_definitions(jq PRIVATE ${DEFS})
if(MSVC)
	# MSVC 2019 C/C++ conformant preprocessor - https://devblogs.microsoft.com/cppblog/announcing-full-support-for-a-c-c-conformant-preprocessor-in-msvc/
	target_compile_options(jq PRIVATE /Zc:preprocessor)
	target_compile_definitions(jq PRIVATE _CRT_SECURE_NO_WARNINGS _CRT_NONSTDC_NO_WARNINGS)
endif()
foreach(HAVE_FUNC ${HAVE_FUNCS})
	target_compile_definitions(jq PRIVATE ${HAVE_FUNC}=${${HAVE_FUNC}})
endforeach()
target_include_directories(jq PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
target_link_libraries(jq PRIVATE libjq)
if(WIN32)
	target_link_libraries(jq PRIVATE shlwapi)
endif()

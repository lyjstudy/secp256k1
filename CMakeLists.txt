
cmake_minimum_required(VERSION 3.5)
project(secp256k1)

# libsecp256k1 use a different set of flags.
add_compiler_flag(
	-pedantic
	-Wshadow
	-Wno-unused-function
	-Wno-nonnull
	-Wno-overlength-strings
)

add_c_compiler_flag(
	-std=c89
	-Wno-long-long
)

# Default visibility is hidden on all targets.
set(CMAKE_C_VISIBILITY_PRESET hidden)

include_directories(
	.
	src
	# For the config
	${CMAKE_CURRENT_BINARY_DIR}/src
)

# The library
add_library(secp256k1 src/secp256k1.c)
target_include_directories(secp256k1 PUBLIC include)

# We check if amd64 asm is supported.
check_c_source_compiles("
	#include <stdint.h>
	int main() {
		uint64_t a = 11, tmp;
		__asm__ __volatile__(\"movq \$0x100000000,%1; mulq %%rsi\" : \"+a\"(a) : \"S\"(tmp) : \"cc\", \"%rdx\");
		return 0;
	}
" USE_ASM_X86_64)

# We make sure __int128 is defined
include(CheckTypeSize)
check_type_size(__int128 SIZEOF___INT128)
if(SIZEOF___INT128 EQUAL 16)
	set(HAVE___INT128 1)
else()
	# If we do not support __int128, we should be falling back
	# on 32bits implementations for field and scalar.
endif()

# Detect if we are on a 32 or 64 bits plateform and chose
# scalar and filed implementation accordingly
if(CMAKE_SIZEOF_VOID_P EQUAL 8)
	# 64 bits implementationr require either __int128 or asm support.
	if (HAVE___INT128 OR USE_ASM_X86_64)
		set(USE_SCALAR_4X64 1)
		set(USE_FIELD_5X52 1)
	else()
		message(SEND_ERROR "Compiler does not support __int128 or insline assembly")
	endif()
else()
	set(USE_SCALAR_8X32 1)
	set(USE_FIELD_10X26 1)
endif()

set(USE_NUM_NONE 1)
set(USE_FIELD_INV_BUILTIN 1)
set(USE_SCALAR_INV_BUILTIN 1)
set(ENABLE_MODULE_RECOVERY 1)
set(ENABLE_MODULE_ECDH 1)
set(ENABLE_MODULE_MULTISET 1)


# Generate the config
configure_file(src/libsecp256k1-config.h.cmake.in src/libsecp256k1-config.h ESCAPE_QUOTES)
target_compile_definitions(secp256k1 PRIVATE HAVE_CONFIG_H SECP256K1_BUILD)


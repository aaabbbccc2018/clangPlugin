cmake_minimum_required (VERSION 2.6)
project (Printer)

set( CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin )
set( CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib )
set( CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib )

set (LLVM_SRC_DIR ${CLANG_PLUGIN_DIR}/llvm)
set (CLANG_SRC_DIR ${LLVM_SRC_DIR}/tools/clang)
set (LLVM_BUILD_DIR ${CLANG_PLUGIN_DIR}/build.llvm)
set (CLANG_BUILD_DIR ${LLVM_BUILD_DIR}/tools/clang)

add_definitions (-D__STDC_LIMIT_MACROS -D__STDC_CONSTANT_MACROS)
add_definitions (-D_GNU_SOURCE -DHAVE_CLANG_CONFIG_H)

# this sets up the devel clang as our compiler
set (CMAKE_CXX_COMPILER "${LLVM_BUILD_DIR}/bin/clang++")
set (CMAKE_CC_COMPILER "${LLVM_BUILD_DIR}/bin/clang")

set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC -fno-common -Woverloaded-virtual -Wcast-qual -fno-strict-aliasing -pedantic -Wno-long-long -Wall -W -Wno-unused-parameter -Wwrite-strings -fno-exceptions -fno-rtti")
set (CMAKE_MODULE_LINKER_FLAGS "-Wl,-flat_namespace -Wl,-undefined -Wl,suppress")

set (LLVM_LIBS
	LLVMJIT
	LLVMX86CodeGen
	LLVMX86AsmParser
	LLVMX86Disassembler
	LLVMExecutionEngine
	LLVMAsmPrinter
	LLVMSelectionDAG
	LLVMX86AsmPrinter
	LLVMX86Info
	LLVMMCParser
	LLVMCodeGen
	LLVMX86Utils
	LLVMScalarOpts
	LLVMInstCombine
	LLVMTransformUtils
	LLVMipa
	LLVMAnalysis
	LLVMTarget
	LLVMCore
	LLVMMC
	LLVMSupport
)

macro(add_clang_plugin name)
	set (srcs ${ARGN})

	include_directories( "${LLVM_SRC_DIR}/include"
		"${CLANG_SRC_DIR}/include"
		"${LLVM_BUILD_DIR}/include" 
		"${CLANG_BUILD_DIR}/include" )
	link_directories( "${LLVM_BUILD_DIR}/lib" )

	add_library( ${name} SHARED ${srcs} )

	if (SYMBOL_FILE)
		set_target_properties( ${name} PROPERTIES LINK_FlAGS
			"-exported_symbols_list ${SYMBOL_FILE}")
	endif()

	foreach (clang_lib ${CLANG_LIBS})
		target_link_libraries( ${name} ${clang_lib} )
	endforeach()

	foreach (llvm_lib ${LLVM_LIBS})
		target_link_libraries( ${name} ${llvm_lib} )
	endforeach()

	foreach (user_lib ${USER_LIBS})
		target_link_libraries( ${name} ${user_lib} )
	endforeach()

endmacro(add_clang_plugin)


add_custom_target(CopyARCLibraries
    COMMAND ${PYTHON_EXECUTABLE} ${CMAKE_SOURCE_DIR}/scripts/copyARCLibraries "${LLVM_BUILD_DIR}"
)

add_custom_target(XcodeTemplate
    COMMAND ${PYTHON_EXECUTABLE} ${CMAKE_SOURCE_DIR}/scripts/copyTemplate -DCLANG_INSTALL_PATH=${LLVM_BUILD_DIR}/bin/clang -DCLANG_PLUGIN_PATH=${CLANG_PLUGIN_DIR} ${CMAKE_SOURCE_DIR}/Library .
	DEPENDS CopyARCLibraries
)

add_custom_target(xcode-analyzer
    COMMAND ${CLANG_SRC_DIR}/tools/scan-build/set-xcode-analyzer --use-checker-build="${LLVM_BUILD_DIR}/bin/clang"
	DEPENDS CopyARCLibraries
)

add_subdirectory(detectTrailingWS)
add_subdirectory(testChecker)


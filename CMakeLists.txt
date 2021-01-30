cmake_minimum_required(VERSION 3.16)
project(capstone VERSION 5.0.0)

enable_testing()

## Define command-line options for building Capstone.

option(CAPSTONE_BUILD_STATIC_RUNTIME "Embed static runtime" ON)
option(CAPSTONE_BUILD_STATIC "Build static library" ON)
option(CAPSTONE_BUILD_SHARED "Build shared library" ON)
option(CAPSTONE_BUILD_DIET "Build diet library" OFF)
option(CAPSTONE_BUILD_TESTS "Build tests" ON)
option(CAPSTONE_BUILD_CSTOOL "Build cstool" ON)
option(CAPSTONE_USE_DEFAULT_ALLOC "Use default memory allocation functions" ON)
option(CAPSTONE_ARCHITECTURE_DEFAULT "Whether architectures are enabled by default" ON)

option(CAPSTONE_X86_REDUCE "x86 with reduce instruction sets to minimize library" OFF)
option(CAPSTONE_X86_ATT_DISABLE "Disable x86 AT&T syntax" OFF)
option(CAPSTONE_OSXKERNEL_SUPPORT "Support to embed Capstone into OS X Kernel extensions" OFF)

set(supported_architectures ARM ARM64 M68K MIPS PPC SPARC SYSZ XCORE X86 TMS320C64X M680X EVM MOS65XX)
foreach (arch IN LISTS supported_architectures)
    option("CAPSTONE_${arch}_SUPPORT" "${arch} support" ${CAPSTONE_ARCHITECTURE_DEFAULT})
endforeach ()

## Process options

add_library(capstone_config INTERFACE)
target_include_directories(capstone_config INTERFACE $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>)

foreach (arch ${supported_architectures})
    set(option_name "CAPSTONE_${arch}_SUPPORT")
    if (${option_name})
        message(STATUS "Enabling ${option_name}")
        target_compile_definitions(capstone_config INTERFACE ${option_name})
    endif ()
endforeach ()

if (CAPSTONE_BUILD_STATIC_RUNTIME)
    set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif ()

if (CAPSTONE_BUILD_DIET)
    target_compile_definitions(capstone_config INTERFACE CAPSTONE_DIET)
endif ()

if (CAPSTONE_USE_DEFAULT_ALLOC)
    target_compile_definitions(capstone_config INTERFACE CAPSTONE_USE_SYS_DYN_MEM)
endif ()

if (CAPSTONE_X86_REDUCE)
    target_compile_definitions(capstone_config INTERFACE CAPSTONE_X86_REDUCE)
endif ()

if (CAPSTONE_X86_ATT_DISABLE)
    target_compile_definitions(capstone_config INTERFACE CAPSTONE_X86_ATT_DISABLE)
endif ()

if (CAPSTONE_OSXKERNEL_SUPPORT)
    target_compile_definitions(capstone_config INTERFACE CAPSTONE_HAS_OSXKERNEL)
endif ()

## Source lists

set(SOURCES_ENGINE
    cs.c
    MCInst.c
    MCInstrDesc.c
    MCRegisterInfo.c
    SStream.c
    utils.c
    )
set(HEADERS_ENGINE
    cs_priv.h
    LEB128.h
    MathExtras.h
    MCDisassembler.h
    MCFixedLenDisassembler.h
    MCInst.h
    MCInstrDesc.h
    MCRegisterInfo.h
    SStream.h
    utils.h
    )
set(HEADERS_COMMON
    include/capstone/arm64.h
    include/capstone/arm.h
    include/capstone/capstone.h
    include/capstone/evm.h
    include/capstone/mips.h
    include/capstone/ppc.h
    include/capstone/x86.h
    include/capstone/sparc.h
    include/capstone/systemz.h
    include/capstone/xcore.h
    include/capstone/m68k.h
    include/capstone/tms320c64x.h
    include/capstone/m680x.h
    include/capstone/mos65xx.h
    include/capstone/platform.h
    )

source_group("Source\\Engine" FILES ${SOURCES_ENGINE})
source_group("Include\\Common" FILES ${HEADERS_COMMON})
source_group("Include\\Engine" FILES ${HEADERS_ENGINE})

## Create capstone target

add_library(capstone ${SOURCES_ENGINE} ${HEADERS_COMMON} ${HEADERS_ENGINE})
add_library(capstone::capstone ALIAS capstone)
set_target_properties(capstone PROPERTIES
                      OUTPUT_NAME capstone
                      IMPORT_SUFFIX _dll.lib
                      VERSION ${PROJECT_VERSION}
                      SOVERSION ${PROJECT_VERSION_MAJOR})

target_link_libraries(capstone PUBLIC capstone_config)
target_include_directories(capstone INTERFACE $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>)
target_compile_definitions(capstone PRIVATE $<$<STREQUAL:$<TARGET_PROPERTY:TYPE>,SHARED_LIBRARY>:CAPSTONE_SHARED>)

## Add architecture modules

function(add_architecture arch)
    # Parse arguments and set defaults
    cmake_parse_arguments("" "" "DEFINE" "SOURCES;HEADERS;TESTS" ${ARGN})
    if (NOT _TESTS)
        string(TOLOWER "test_${arch}.c" _TESTS)
    endif ()
    if (NOT _DEFINE)
        set(_DEFINE "CAPSTONE_HAS_${arch}")
    endif ()

    # Bail out if architecture disabled
    if (NOT CAPSTONE_${arch}_SUPPORT)
        return()
    endif ()

    # Add arch objects
    add_library(capstone_${arch} OBJECT ${_SOURCES} ${_HEADERS})
    set_target_properties(capstone_${arch} PROPERTIES FOLDER "Arch/${arch}")
    target_link_libraries(capstone_${arch} PRIVATE capstone_config)

    # Update config
    target_compile_definitions(capstone_config INTERFACE ${_DEFINE})

    # Add arch to capstone
    target_link_libraries(capstone PUBLIC $<BUILD_INTERFACE:capstone_${arch}>)

    # Add tests for arch
    if (CAPSTONE_BUILD_TESTS)
        foreach (test IN LISTS _TESTS)
            string(REGEX REPLACE ".c$" "" test_name ${test})
            add_executable(${test_name} "tests/${test}")
            target_link_libraries(${test_name} PRIVATE capstone::capstone)
            add_test(NAME "capstone_${test_name}" COMMAND ${test_name})
        endforeach ()
    endif ()
endfunction()

add_architecture(ARM
                 SOURCES
                 arch/ARM/ARMDisassembler.c
                 arch/ARM/ARMInstPrinter.c
                 arch/ARM/ARMMapping.c
                 arch/ARM/ARMModule.c
                 HEADERS
                 arch/ARM/ARMAddressingModes.h
                 arch/ARM/ARMBaseInfo.h
                 arch/ARM/ARMDisassembler.h
                 arch/ARM/ARMGenAsmWriter.inc
                 arch/ARM/ARMGenDisassemblerTables.inc
                 arch/ARM/ARMGenInstrInfo.inc
                 arch/ARM/ARMGenRegisterInfo.inc
                 arch/ARM/ARMGenSubtargetInfo.inc
                 arch/ARM/ARMInstPrinter.h
                 arch/ARM/ARMMapping.h
                 )

add_architecture(ARM64
                 SOURCES
                 arch/AArch64/AArch64BaseInfo.c
                 arch/AArch64/AArch64Disassembler.c
                 arch/AArch64/AArch64InstPrinter.c
                 arch/AArch64/AArch64Mapping.c
                 arch/AArch64/AArch64Module.c
                 HEADERS
                 arch/AArch64/AArch64AddressingModes.h
                 arch/AArch64/AArch64BaseInfo.h
                 arch/AArch64/AArch64Disassembler.h
                 arch/AArch64/AArch64GenAsmWriter.inc
                 arch/AArch64/AArch64GenDisassemblerTables.inc
                 arch/AArch64/AArch64GenInstrInfo.inc
                 arch/AArch64/AArch64GenRegisterInfo.inc
                 arch/AArch64/AArch64GenSubtargetInfo.inc
                 arch/AArch64/AArch64InstPrinter.h
                 arch/AArch64/AArch64Mapping.h
                 )

add_architecture(MIPS
                 SOURCES
                 arch/Mips/MipsDisassembler.c
                 arch/Mips/MipsInstPrinter.c
                 arch/Mips/MipsMapping.c
                 arch/Mips/MipsModule.c
                 HEADERS
                 arch/Mips/MipsDisassembler.h
                 arch/Mips/MipsGenAsmWriter.inc
                 arch/Mips/MipsGenDisassemblerTables.inc
                 arch/Mips/MipsGenInstrInfo.inc
                 arch/Mips/MipsGenRegisterInfo.inc
                 arch/Mips/MipsGenSubtargetInfo.inc
                 arch/Mips/MipsInstPrinter.h
                 arch/Mips/MipsMapping.h
                 )

add_architecture(PPC
                 DEFINE CAPSTONE_HAS_POWERPC
                 SOURCES
                 arch/PowerPC/PPCDisassembler.c
                 arch/PowerPC/PPCInstPrinter.c
                 arch/PowerPC/PPCMapping.c
                 arch/PowerPC/PPCModule.c
                 HEADERS
                 arch/PowerPC/PPCDisassembler.h
                 arch/PowerPC/PPCGenAsmWriter.inc
                 arch/PowerPC/PPCGenDisassemblerTables.inc
                 arch/PowerPC/PPCGenInstrInfo.inc
                 arch/PowerPC/PPCGenRegisterInfo.inc
                 arch/PowerPC/PPCGenSubtargetInfo.inc
                 arch/PowerPC/PPCInstPrinter.h
                 arch/PowerPC/PPCMapping.h
                 arch/PowerPC/PPCPredicates.h
                 )

add_architecture(X86
                 TESTS test_x86.c test_customized_mnem.c
                 SOURCES
                 arch/X86/X86Disassembler.c
                 arch/X86/X86DisassemblerDecoder.c
                 arch/X86/X86IntelInstPrinter.c
                 arch/X86/X86Mapping.c
                 arch/X86/X86Module.c
                 $<$<NOT:$<BOOL:${CAPSTONE_BUILD_DIET}>>:arch/X86/X86ATTInstPrinter.c>
                 HEADERS
                 arch/X86/X86BaseInfo.h
                 arch/X86/X86Disassembler.h
                 arch/X86/X86DisassemblerDecoder.h
                 arch/X86/X86DisassemblerDecoderCommon.h
                 arch/X86/X86GenAsmWriter.inc
                 arch/X86/X86GenAsmWriter1.inc
                 arch/X86/X86GenAsmWriter1_reduce.inc
                 arch/X86/X86GenAsmWriter_reduce.inc
                 arch/X86/X86GenDisassemblerTables.inc
                 arch/X86/X86GenDisassemblerTables_reduce.inc
                 arch/X86/X86GenInstrInfo.inc
                 arch/X86/X86GenInstrInfo_reduce.inc
                 arch/X86/X86GenRegisterInfo.inc
                 arch/X86/X86InstPrinter.h
                 arch/X86/X86Mapping.h
                 )

add_architecture(SPARC
                 SOURCES
                 arch/Sparc/SparcDisassembler.c
                 arch/Sparc/SparcInstPrinter.c
                 arch/Sparc/SparcMapping.c
                 arch/Sparc/SparcModule.c
                 HEADERS
                 arch/Sparc/Sparc.h
                 arch/Sparc/SparcDisassembler.h
                 arch/Sparc/SparcGenAsmWriter.inc
                 arch/Sparc/SparcGenDisassemblerTables.inc
                 arch/Sparc/SparcGenInstrInfo.inc
                 arch/Sparc/SparcGenRegisterInfo.inc
                 arch/Sparc/SparcGenSubtargetInfo.inc
                 arch/Sparc/SparcInstPrinter.h
                 arch/Sparc/SparcMapping.h
                 arch/Sparc/SparcMappingInsn.inc
                 )

add_architecture(SYSZ
                 TESTS test_systemz.c
                 SOURCES
                 arch/SystemZ/SystemZDisassembler.c
                 arch/SystemZ/SystemZInstPrinter.c
                 arch/SystemZ/SystemZMapping.c
                 arch/SystemZ/SystemZModule.c
                 arch/SystemZ/SystemZMCTargetDesc.c
                 HEADERS
                 arch/SystemZ/SystemZDisassembler.h
                 arch/SystemZ/SystemZGenAsmWriter.inc
                 arch/SystemZ/SystemZGenDisassemblerTables.inc
                 arch/SystemZ/SystemZGenInsnNameMaps.inc
                 arch/SystemZ/SystemZGenInstrInfo.inc
                 arch/SystemZ/SystemZGenRegisterInfo.inc
                 arch/SystemZ/SystemZGenSubtargetInfo.inc
                 arch/SystemZ/SystemZInstPrinter.h
                 arch/SystemZ/SystemZMapping.h
                 arch/SystemZ/SystemZMappingInsn.inc
                 arch/SystemZ/SystemZMCTargetDesc.h
                 )

add_architecture(XCORE
                 SOURCES
                 arch/XCore/XCoreDisassembler.c
                 arch/XCore/XCoreInstPrinter.c
                 arch/XCore/XCoreMapping.c
                 arch/XCore/XCoreModule.c
                 HEADERS
                 arch/XCore/XCoreDisassembler.h
                 arch/XCore/XCoreGenAsmWriter.inc
                 arch/XCore/XCoreGenDisassemblerTables.inc
                 arch/XCore/XCoreGenInstrInfo.inc
                 arch/XCore/XCoreGenRegisterInfo.inc
                 arch/XCore/XCoreInstPrinter.h
                 arch/XCore/XCoreMapping.h
                 arch/XCore/XCoreMappingInsn.inc
                 )

add_architecture(M68K
                 SOURCES
                 arch/M68K/M68KDisassembler.c
                 arch/M68K/M68KInstPrinter.c
                 arch/M68K/M68KModule.c
                 HEADERS
                 arch/M68K/M68KDisassembler.h
                 )

add_architecture(TMS320C64X
                 SOURCES
                 arch/TMS320C64x/TMS320C64xDisassembler.c
                 arch/TMS320C64x/TMS320C64xInstPrinter.c
                 arch/TMS320C64x/TMS320C64xMapping.c
                 arch/TMS320C64x/TMS320C64xModule.c
                 HEADERS
                 arch/TMS320C64x/TMS320C64xDisassembler.h
                 arch/TMS320C64x/TMS320C64xGenAsmWriter.inc
                 arch/TMS320C64x/TMS320C64xGenDisassemblerTables.inc
                 arch/TMS320C64x/TMS320C64xGenInstrInfo.inc
                 arch/TMS320C64x/TMS320C64xGenRegisterInfo.inc
                 arch/TMS320C64x/TMS320C64xInstPrinter.h
                 arch/TMS320C64x/TMS320C64xMapping.h
                 )

add_architecture(M680X
                 SOURCES
                 arch/M680X/M680XDisassembler.c
                 arch/M680X/M680XInstPrinter.c
                 arch/M680X/M680XModule.c
                 HEADERS
                 arch/M680X/M680XInstPrinter.h
                 arch/M680X/M680XDisassembler.h
                 arch/M680X/M680XDisassemblerInternals.h
                 )

add_architecture(EVM
                 SOURCES
                 arch/EVM/EVMDisassembler.c
                 arch/EVM/EVMInstPrinter.c
                 arch/EVM/EVMMapping.c
                 arch/EVM/EVMModule.c
                 HEADERS
                 arch/EVM/EVMDisassembler.h
                 arch/EVM/EVMInstPrinter.h
                 arch/EVM/EVMMapping.h
                 arch/EVM/EVMMappingInsn.inc
                 )

add_architecture(MOS65XX
                 SOURCES
                 arch/MOS65XX/MOS65XXModule.c
                 arch/MOS65XX/MOS65XXDisassembler.c
                 HEADERS
                 arch/MOS65XX/MOS65XXDisassembler.h
                 )

## Testing

if (CAPSTONE_BUILD_TESTS)
    # Add general tests
    add_executable(test_basic tests/test_basic.c)
    target_link_libraries(test_basic PRIVATE capstone::capstone)
    add_test(NAME capstone_test_basic COMMAND test_basic)

    add_executable(test_detail tests/test_detail.c)
    target_link_libraries(test_detail PRIVATE capstone::capstone)
    add_test(NAME capstone_test_detail COMMAND test_detail)

    add_executable(test_skipdata tests/test_skipdata.c)
    target_link_libraries(test_skipdata PRIVATE capstone::capstone)
    add_test(NAME capstone_test_skipdata COMMAND test_skipdata)

    add_executable(test_iter tests/test_iter.c)
    target_link_libraries(test_iter PRIVATE capstone::capstone)
    add_test(NAME capstone_test_iter COMMAND test_iter)

    if (CAPSTONE_ARM_SUPPORT)
        set(ARM_REGRESS_TEST test_arm_regression.c)
        STRING(REGEX REPLACE ".c$" "" ARM_REGRESS_BIN ${ARM_REGRESS_TEST})
        add_executable(${ARM_REGRESS_BIN} "suite/arm/${ARM_REGRESS_TEST}")
        target_link_libraries(${ARM_REGRESS_BIN} PRIVATE capstone::capstone)
        add_test(NAME "capstone_${ARM_REGRESS_BIN}" COMMAND ${ARM_REGRESS_BIN})
    endif ()

    # fuzz target built with the tests
    add_executable(fuzz_disasm suite/fuzz/onefile.c suite/fuzz/fuzz_disasm.c)
    target_link_libraries(fuzz_disasm PRIVATE capstone::capstone)
endif ()

## Installation

include(GNUInstallDirs)
set(CAPSTONE_INSTALL_CMAKEDIR "${CMAKE_INSTALL_LIBDIR}/cmake/capstone"
    CACHE STRING "Install destination for CMake package scripts")

install(TARGETS capstone capstone_config
        EXPORT capstone
        INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")
install(FILES ${HEADERS_COMMON}
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/capstone)

configure_file(capstone.pc.in ${CMAKE_CURRENT_BINARY_DIR}/capstone.pc @ONLY)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/capstone.pc
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)

install(EXPORT capstone
        DESTINATION ${CAPSTONE_INSTALL_CMAKEDIR}
        NAMESPACE capstone::
        FILE capstone-config.cmake)

if (BUILD_SHARED_LIBS AND CAPSTONE_BUILD_CSTOOL)
    FILE(GLOB CSTOOL_SRC cstool/*.c)
    add_executable(cstool ${CSTOOL_SRC})
    target_link_libraries(cstool PRIVATE capstone::capstone)

    install(TARGETS cstool)
endif ()

add_library(lzma STATIC
  include/7zTypes.h
  include/7zWindows.h
  include/Alloc.h
  include/Bra.h
  include/Compiler.h
  include/CpuArch.h
  include/Delta.h
  include/LzFind.h
  include/LzHash.h
  include/Lzma86.h
  include/LzmaDec.h
  include/LzmaEnc.h
  include/Precomp.h
  include/RotateDefs.h
  include/Sort.h
  src/Alloc.c
  src/Bra.c
  src/Bra86.c
  src/BraIA64.c
  src/CpuArch.c
  src/Delta.c
  src/LzFind.c
  src/Lzma86Dec.c
  src/LzmaDec.c
  src/LzmaEnc.c
  src/Sort.c
)

target_compile_definitions(lzma PRIVATE Z7_ST)

target_include_directories(lzma PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}/include")
target_include_directories(lzma INTERFACE "${CMAKE_CURRENT_SOURCE_DIR}/include")

set_target_properties(lzma PROPERTIES POSITION_INDEPENDENT_CODE ON)

option(WITH_LZMA_ASM "Use lzma asm" ON)

if(WITH_LZMA_ASM)

    # Detect
    include(CheckSymbolExists)
    check_symbol_exists("__aarch64__" "" CPU_ARM64)
    check_symbol_exists("_M_AMD64" "" MSVC_X64)
    check_symbol_exists("__x86_64__" "" CPU_X64)

    # ------------------------------------------------------------
    # Linux
    # ------------------------------------------------------------
    if(CMAKE_SYSTEM_NAME STREQUAL "Linux")

        if(CPU_ARM64 OR CPU_X64)

            enable_language(ASM)

            find_program(UASM_EXECUTABLE NAMES uasm64 uasm REQUIRED)
            set(CMAKE_ASM_COMPILER ${UASM_EXECUTABLE} CACHE FILEPATH "" FORCE)

            set_source_files_properties(
                src/LzmaDec.c
                PROPERTIES COMPILE_DEFINITIONS Z7_LZMA_DEC_OPT
            )

            # -------- ARM64 --------
            if(CPU_ARM64)

                message(STATUS "Using ARM64 ASM for LZMA (Linux)")

                set(ASM_SRC ${CMAKE_CURRENT_SOURCE_DIR}/Asm/x86/LzmaDecOpt.S)
                set(ASM_OBJ ${CMAKE_CURRENT_BINARY_DIR}/LzmaDecOpt.S.obj)

            # -------- x86_64 --------
            elseif(CPU_X64)

                message(STATUS "Using x86_64 ASM for LZMA (Linux)")

                set(ASM_SRC ${CMAKE_CURRENT_SOURCE_DIR}/Asm/x86/LzmaDecOpt.asm)
                set(ASM_OBJ ${CMAKE_CURRENT_BINARY_DIR}/LzmaDecOpt.asm.obj)

            endif()

            add_custom_command(
                OUTPUT ${ASM_OBJ}
                COMMAND ${UASM_EXECUTABLE}
                        -elf64 -DABI_LINUX -nologo
                        -Fo ${ASM_OBJ}
                        ${ASM_SRC}
                DEPENDS ${ASM_SRC}
                COMMENT "Assembling LzmaDecOpt with UASM"
            )

            target_sources(lzma PRIVATE ${ASM_OBJ})

        endif()

    # ------------------------------------------------------------
    # Windows
    # ------------------------------------------------------------
    elseif(WIN32)

        # -------- MSVC x64 --------
        if(MSVC AND MSVC_X64)

            message(STATUS "Using MASM (MSVC) for x86_64 LZMA")

            enable_language(ASM_MASM)

            set_source_files_properties(
                src/LzmaDec.c
                PROPERTIES COMPILE_DEFINITIONS Z7_LZMA_DEC_OPT
            )

            target_sources(lzma PRIVATE Asm/x86/LzmaDecOpt.asm)
            set_source_files_properties(
                Asm/x86/LzmaDecOpt.asm
                PROPERTIES LANGUAGE ASM_MASM
            )

        # -------- MinGW x64 (UASM) --------
        elseif(MINGW AND CPU_X64)

            message(STATUS "Using UASM (MinGW/MSYS2) for x86_64 LZMA")

            find_program(UASM_EXECUTABLE NAMES uasm64 uasm REQUIRED)
            set(CMAKE_ASM_COMPILER ${UASM_EXECUTABLE} CACHE FILEPATH "" FORCE)

            set_source_files_properties(
                src/LzmaDec.c
                PROPERTIES COMPILE_DEFINITIONS Z7_LZMA_DEC_OPT
            )

            set(ASM_SRC ${CMAKE_CURRENT_SOURCE_DIR}/Asm/x86/LzmaDecOpt.asm)
            set(ASM_OBJ ${CMAKE_CURRENT_BINARY_DIR}/LzmaDecOpt.asm.obj)

            add_custom_command(
                OUTPUT ${ASM_OBJ}
                COMMAND ${UASM_EXECUTABLE}
                        -win64 -nologo
                        -Fo ${ASM_OBJ}
                        ${ASM_SRC}
                DEPENDS ${ASM_SRC}
                COMMENT "Assembling LzmaDecOpt with UASM"
            )

            target_sources(lzma PRIVATE ${ASM_OBJ})

        endif()
    endif()
endif()

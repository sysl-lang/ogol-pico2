cmake_minimum_required(VERSION 3.13)

set(PICO_BOARD pico2_w CACHE STRING "Board type")

# **The RP2350 has two personalities and this program runs on either.** The chip carries a pair of
# Cortex-M33s and a pair of Hazard3 RISC-V cores, and which set boots is a build-time choice rather
# than anything the board does — so this is one project with two configurations, not two projects.
#
#     cmake -B build                                     the Cortex-M33s
#     cmake -B build-riscv -DPICO_PLATFORM=rp2350-riscv \
#           -DPICO_TOOLCHAIN_PATH=/path/to/riscv-toolchain-15
#
# The RISC-V configuration needs a toolchain of its own; the SDK looks for `riscv32-unknown-elf` and
# three other triples, none of which the Arm toolchain provides.
set(PICO_PLATFORM rp2350 CACHE STRING "rp2350 for the Cortex-M33s, rp2350-riscv for the Hazard3s")

# **sysl has to be told the same thing, and nothing derives it for us**: `sysl build-c` is a program
# CMake knows nothing about, so the archive would otherwise be compiled for whichever core this file
# happened to name. Getting it wrong is not a link error on the sysl side — it is an image that
# faults at whatever executes first.
if(PICO_PLATFORM STREQUAL "rp2350-riscv")
    set(SYSL_TARGET riscv32-freestanding)
else()
    set(SYSL_TARGET thumb-freestanding-softfp)
endif()

# No `PICO_HARD_FLOAT_ABI`: `thumb-freestanding-softfp` is what lets this be a stock pico-sdk, with
# sysl following the SDK's default rather than the SDK following sysl. See `repl/CMakeLists.txt`,
# which carries the whole of that story. The question does not arise on the Hazard3 at all — it is
# RV32IMAC with no F extension, so there are no floating registers to disagree about.

include(pico_sdk_import.cmake)

project(sysl_ogol C CXX ASM)

pico_sdk_init()

set(SYSL_ARCHIVE ${CMAKE_CURRENT_BINARY_DIR}/libogol.a)

# **Both the language and the board are ordinary coordinates**, declared in `ogol/package.hocon`, so
# this project builds from a fresh clone. It did not until `sysl-lang/ogol` was tagged v0.1.0 on
# 2026-08-10; before that the language came by `--lib` from a checkout beside this one.
#
# It was a second `--lib` until sysl 0.0.37, and not by choice — card `0055`. A `--lib` source root
# binding a top-level name silently shadowed the sub-modules of a **coordinate** dependency binding
# the same one, and every package in this org binds `sh`, so `--lib ogol` beside
# `pico2 { git = … }` reported `no module is called 'sh.sysl.pico2.externs'` from inside the cached
# package. **This needs sysl 0.0.37 or newer**; an older one wants the coordinate turned back into a
# second `--lib`.
# **`pico2` carries C now, and it needs the SDK's headers to compile.**
#
# This project needed none of the following while it named pico2 v0.0.4, because that version was
# pure declaration — every name in it was a symbol the SDK already had. v0.0.5 onwards carries
# `scan.c` and `dns.c` for the radio, and `sysl build-c` is a program CMake knows nothing about, so
# what it can see has to be handed to it. `wifi/CMakeLists.txt` carries the whole of this story.
#
# The libc half comes from the toolchain rather than being written down: `-print-sysroot` answers with
# the directory whose `include` holds newlib's headers, and the obvious path one level up has no
# `string.h` in it.
execute_process(
    COMMAND ${CMAKE_C_COMPILER} -print-sysroot
    OUTPUT_VARIABLE ARM_SYSROOT
    OUTPUT_STRIP_TRAILING_WHITESPACE)

# The SDK half is asked of the SDK: it generates some of its headers into its own build directory, so
# a fixed list written here would go stale the first time it moved one. What is asked is the
# **executable's** resolved list rather than any one library's, because `cyw43.h` includes
# `lwip/netif.h`, which is two link libraries away — reading `INCLUDE_DIRECTORIES` off the target gets
# the transitive closure CMake has already worked out for the C side. It names a target defined
# further down, which is fine: a generator expression is resolved after the whole file is read.
set(SYSL_C_INCLUDES
    ${ARM_SYSROOT}/include
    $<TARGET_PROPERTY:sysl_ogol,INCLUDE_DIRECTORIES>)

# And the macros, which finding the headers does not supply: `pico/cyw43_arch.h` `#error`s rather than
# picking an architecture variant, so every include path can be right and the compile still stops
# inside a header.
set(SYSL_C_DEFINES $<TARGET_PROPERTY:sysl_ogol,COMPILE_DEFINITIONS>)

add_custom_command(
    OUTPUT ${SYSL_ARCHIVE} ${SYSL_ARCHIVE}.h
    COMMAND sysl build-c ${CMAKE_CURRENT_SOURCE_DIR}/ogol --target ${SYSL_TARGET}
            "$<$<BOOL:${SYSL_C_INCLUDES}>:--include-path;$<JOIN:${SYSL_C_INCLUDES},;--include-path;>>"
            "$<$<BOOL:${SYSL_C_DEFINES}>:-D;$<JOIN:${SYSL_C_DEFINES},;-D;>>"
            -o ${SYSL_ARCHIVE}
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/ogol/main.sysl
            ${CMAKE_CURRENT_SOURCE_DIR}/ogol/package.hocon
    COMMENT "Compiling Ogol and its console for ${SYSL_TARGET}"
    # **`COMMAND_EXPAND_LISTS`, without which the two generator expressions above are useless.** A
    # quoted argument that expands to a `;`-separated list stays *one* argument unless this is given,
    # so `sysl` is handed a single `--include-path;/a;--include-path;/b` and answers
    # `Unknown option`. Unquoting is not the fix either — CMake then splits the variable at parse
    # time and the `$<` never evaluates at all.
    COMMAND_EXPAND_LISTS
    VERBATIM)

add_custom_target(sysl_archive DEPENDS ${SYSL_ARCHIVE})

# There is no C in this project at all: sysl exports `main`, the SDK's `crt0` branches straight into
# it, and the archive in the source list is what satisfies CMake's "a target must have sources" rule
# with no translation unit to infer a language from.
add_executable(sysl_ogol ${SYSL_ARCHIVE})

set_target_properties(sysl_ogol PROPERTIES LINKER_LANGUAGE C)

add_dependencies(sysl_ogol sysl_archive)

target_link_libraries(sysl_ogol pico_stdlib pico_cyw43_arch_none ${SYSL_ARCHIVE})

pico_enable_stdio_usb(sysl_ogol 1)
pico_enable_stdio_uart(sysl_ogol 0)

pico_add_extra_outputs(sysl_ogol)

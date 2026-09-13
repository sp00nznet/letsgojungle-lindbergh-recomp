cmake_minimum_required(VERSION 3.20)
project(letsgojungle VERSION 0.1.0 LANGUAGES C)

# Let's Go Jungle (Sega, 2006), statically recompiled.
#
# The game target only exists once you have lifted an ELF, because the runtime
# needs two headers generated from it. Without them this still configures and
# builds the toolkit's own self-check, so the repo is testable out of the box.
#
#   py -3.11 -m tools recomp main.elf ..\games\letsgojungle\generated
#   cmake -S . -B build -A Win32
#   cmake --build build --config Release

add_subdirectory(lindberghrecomp)

set(GEN "${CMAKE_CURRENT_SOURCE_DIR}/games/letsgojungle/generated")

if(EXISTS "${GEN}/recomp_funcs.c" AND EXISTS "${GEN}/recomp_imports.h")
    file(GLOB LIFTED "${GEN}/recomp_funcs*.c")
    add_executable(letsgojungle
        games/letsgojungle/src/host.c
        ${LIFTED}
    )
    lindbergh_runtime(letsgojungle "${GEN}")
    message(STATUS "letsgojungle: building against lifted code in ${GEN}")
else()
    message(STATUS
        "letsgojungle: no lifted code in ${GEN} yet, building the toolkit "
        "self-check only. See the README.")
endif()

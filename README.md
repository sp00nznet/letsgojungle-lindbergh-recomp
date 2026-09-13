# letsgojungle-lindbergh-recomp

**Let's Go Jungle! Lost on the Island of Spice (Sega, 2006) — a Lindbergh
light-gun rail shooter, statically recompiled from its x86 ELF to native C.**

Two people, two guns, one boat, and an island where the insects have opinions.
*Let's Go Jungle* is the best first target on the platform: it is the smallest
Lindbergh disc we have at 1.0 GB, it never left the arcade, and it wants
nothing from the outside world — no network, no card reader, no linked
cabinets. A gun, a trigger, and a rail.

Built on [**lindberghrecomp**](https://github.com/sp00nznet/lindberghrecomp),
the Sega Lindbergh recompilation toolkit, vendored here as a git submodule.

> **No game data here.** No ELF, no disc, no filesystem image, no keys. The
> `.gitignore` refuses all of it. Bring a game tree you can already read; the
> recompiled C is output you generate.

## Status — the whole game lifts

`lgj_final` is a 12.7 MB `ET_EXEC` / `EM_386` binary with entry point
`0x08072d70`, and **it ships its full symbol table**. That is the single best
thing about this target:

```
$ py -3.11 -m tools elf lgj_final
entry      0x08072d70
image      0x08048000 + 0xc23f44
segments   2 PT_LOAD
functions  31752 sized STT_FUNC symbols
imports    406 PLT stubs
  needs    libCg.so  libCgGL.so  libxerces-c.so.26  libGLU.so.1  libGL.so.1
  needs    libsegaapi.so  libpthread.so.0  libm.so.6  libgcc_s.so.1
  needs    libc.so.6  libXext.so.6  libX11.so.6  libdl.so.2
```

No function discovery, no bounds file to export from Ghidra, no heuristic
carving. The ELF says where all 31,752 functions start and how long each one
is. And **all 31,752 lift, in 28 seconds**, into 1,702,616 lines of C. Not one
failed outright.

| | |
|---|---|
| Functions recovered | **31,752** — from the binary's own symbol table |
| Functions lifted | **31,752 / 31,752**, in 28 s |
| C emitted | 1,702,616 lines |
| Instruction coverage | **99.91%** — 1,558 lines are `/* TODO */ abort()` |
| Imports to implement | **406** across 13 libraries |
| Compiles | **Yes** — the 600 most SSE-dense functions, 85,402 SSE instructions, clean MSVC object |
| Runs | Not yet. The 406 imports still need bodies. |

### SSE is done

The first full lift came out at 90.6% coverage, and the missing 9.4% was one
family: *Let's Go Jungle* is a Pentium 4 game that keeps its floats in XMM
registers, and the lifter came from Pentium III targets with no SSE at all.
`movss` alone was 46% of the whole gap.

That got fixed **upstream in
[pcrecomp](https://github.com/sp00nznet/pcrecomp)** rather than here — one x86
lifter serves every PC-era target, and forking it to fix one game is how you
end up maintaining four. Scalar SSE, the compares, the conversions, the 128-bit
moves and bitwise ops, `cmovcc`, and the prefetch hints as no-ops:

| | before | after |
|---|---:|---:|
| Instruction coverage | 90.55% | **99.91%** |
| `/* TODO */ abort()` lines | 160,820 | **1,558** |

And the output compiles: the 600 densest SSE functions in the game — 85,402 SSE
instructions between them — build to a clean 10 MB object with no warnings.

What is still unlifted is 1,558 lines of x87 leftovers, MMX, packed SSE
arithmetic, and 83 port-I/O instructions that userspace has no business
executing anyway.

### What is left

**The 406 imports.** Shorter than it looks: stock glibc, stock OpenGL/GLU,
stock X11, NVIDIA's Cg shader runtime, Xerces — and exactly one Sega library,
`libsegaapi.so`, the sound API. Everything else is a library that still exists
and whose behaviour is documented. `hle_call()` aborts naming whatever it wants
next, so the order of work picks itself: glibc, then GL and Cg, then sound,
then the JVS I/O the guns arrive on.

### The disc

The retail DVD is encrypted and
[lindberghrecomp does not decrypt it](https://github.com/sp00nznet/lindberghrecomp/blob/master/docs/disc-format.md) —
the dump carves to named payload files and every one is ciphertext under a key
that lives in the cabinet. Bring a game tree you can already read; arcade
preservation projects have published clean dumps of many Lindbergh titles,
taken from original DVDs and cabinet hard discs by people who had the keys.
A clean tree looks like this:

```
disk0/
  lgj_final              the game (this is the ELF)
  game                   the launcher script
  lgjrc                  its config
  data/                  ADX/AIX audio, CSB banks, .xaf sound banks
  shader/Cg  extraShader/Cg
  libCg.so  libCgGL.so  libCgFX.so  libpng.so  libxerces-c.so
```

## Reproducing it

```powershell
git clone --recursive https://github.com/sp00nznet/letsgojungle-lindbergh-recomp
cd letsgojungle-lindbergh-recomp

# what the game asks the board for
py -3.11 -m tools elf path\to\main.elf          # run from lindberghrecomp\

# lift it
py -3.11 -m tools recomp path\to\main.elf ..\games\letsgojungle\generated

# build - 32-bit, and the CMake will not let you forget
cmake -S . -B build -A Win32
cmake --build build --config Release
.\build\Release\letsgojungle.exe path\to\main.elf
```

It will then abort on the first library function it wants, naming it. That is
the intended first run and the whole work plan: `hle_call()` prints what is
missing, you write it, you run it again. glibc first, then OpenGL, then sound,
then the JVS I/O the guns arrive on.

## Layout

```
lindberghrecomp/                 the toolkit (git submodule)
games/letsgojungle/
  src/host.c                     load the ELF, enter it, get out of the way
  generated/                     the lifted C (you generate it; gitignored)
```

## Legal

The host and the recompilation toolchain are original work, MIT-licensed.
**No game data, no disc images, no keys, and no circumvention of anything.**
*Let's Go Jungle! Lost on the Island of Spice* is © Sega; this is an
independent, non-commercial preservation project. Built on
[**lindberghrecomp**](https://github.com/sp00nznet/lindberghrecomp).

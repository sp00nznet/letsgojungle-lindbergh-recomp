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

## Status — it renders

**Let's Go Jungle runs its attract mode as recompiled C, at roughly 32 frames
per second.** No emulator, no interpreter: 31,759 functions lifted from the
game's own ELF to C, compiled into a native 32-bit executable, drawing on the
host GPU through the real OpenGL driver.

```
> $env:TEA_DIR = "X:\path\to\disk0"
> .\build\Release\letsgojungle.exe X:\path\to\disk0\lgj_final
[gl] 96 of 96 entry points bound
[crt] main at 0x08411ff0 (argc=1)
[pthread] created thread at 0x084f9d82, 1024 KB stack   ×3
[window] 1360x768
[glX] context created (pixel format 11)
[glX] current: NVIDIA GeForce RTX 5070/PCIe/SSE2 / 4.6.0 NVIDIA 595.97
[cg] indexed 197 precompiled shaders
```

Measured over 60 seconds of attract mode:

| | per run | per frame |
|---|---:|---:|
| `glXSwapBuffers` | 1,939 | — |
| `glClear` | 21,320 | ~11 |
| `glBegin` | 104,319 | ~54 |
| `glBindTexture` | 106,550 | ~55 |
| `glProgramStringARB` | 189 | — |

| | |
|---|---|
| Functions lifted | **31,759 / 31,759**, 28 s, 1.7 M lines of C |
| Instruction coverage | **99.96%** — 622 `RECOMP_TODO` lines, none reached |
| Imports bound | 406 of 406 on every path reached |
| Threads | 3 guest threads, each on its own CPU and stack |
| Shaders | 189 ARB programs uploaded |

`TEA_DIR` must point at the game directory — the engine finds all its data
through it, exactly as the cabinet's launcher script sets it.

### The five things the engine needed

**`XFree86-VidModeExtension`.** It will not build a window without one, and
`libXxf86vm` is linked statically *into* the binary — so there was no import to
bind. The toolkit grew guest-function overrides for it: replace a lifted
function with a host body, by symbol name.

**`Display` is a real struct.** `struct _XDisplay` is public in `Xlib.h` and
`DefaultScreen(dpy)` is a macro that reads offset 132 directly.

**547 GL entry points, not 96.** `es::glh_helper::init_extensions` resolves
them through `glXGetProcAddressARB` and sets its feature flags only if they all
arrive. The 451 with no import get a synthetic address forwarded to the host
driver through a thunk that saves `esp` across the call — which makes the
argument count, unknowable for an arbitrary name, not matter.

**Cg, answered from the disc.** The engine compiles ~260 shader programs at
startup and there is no Cg runtime here. But every shader ships twice, `vs.cg`
beside `vs.asm_gl`, the second being the first as `cgc` compiled it in 2005.
Matching them takes two keys, because neither alone is unique: the `#define`
permutation (which the engine passes as args, and which `cgc` stamped into each
`.asm_gl` header) and the shader itself (whose body, reduced to bare code, is
findable inside the 84 KB preprocessed source the engine hands over).

**`fscanf`.** The game reads its configuration with `while (!feof(f))
fscanf(...)`. An unbound `fscanf` consumes nothing, so it span a hundred
thousand times a second and drew nothing.

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

Those `.so` files are themselves 32-bit x86 ELFs with symbols — `libCg.so` has
284 functions, `libxerces-c.so` has 9,010 — so the 20 Cg and Xerces imports do
not need reimplementing either. They can go through the same pipeline.

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

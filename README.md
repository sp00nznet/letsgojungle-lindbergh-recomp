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

## Status — it opens a window, uploads shaders, and loads textures

`lgj_final` is a 12.7 MB `ET_EXEC` / `EM_386` binary that **ships its full
symbol table** — 31,759 functions with names and sizes, so there is no
function-discovery problem at all. All of them lift, in 28 seconds, into 80
translation units that compile and link into a 25 MB native executable.

That executable runs the game's entire C runtime and a good deal of its engine:

```
> letsgojungle.exe lgj_final
[vidmode] 11 entry points overridden
[gl] 96 of 96 entry points bound
[crt] main at 0x08411ff0 (argc=1)
[pthread] created thread at 0x084f9d82, 1024 KB stack   ×3
[window] 1360x768
[glX] context created (pixel format 11)
[glX] current: NVIDIA GeForce RTX 5070/PCIe/SSE2 / 4.6.0 NVIDIA 595.97
[cg] indexed 197 precompiled shaders
```

| | |
|---|---|
| Functions lifted | **31,759 / 31,759**, 28 s, 1.7 M lines of C |
| Instruction coverage | **99.96%** — 724 `RECOMP_TODO` lines left |
| Boots | CRT, every static constructor, `main` |
| Threads | 3 guest threads, each on its own CPU and stack |
| Window | 1360×768, real WGL context on the host GPU |
| Shaders | **25 ARB programs uploaded** via `glProgramStringARB` |
| Textures | 39 binds, 12 `glTexImage2D` uploads |
| Imports | **0 unbound** on the path reached so far |
| Presents a frame | **Not yet** — stops in `_sShaderManager::getShaderByName` |

### Running it

```powershell
# the engine finds its data through TEA_DIR, exactly as the cabinet's
# launcher script sets it
$env:TEA_DIR = "X:\path\to\disk0"
.\build\Release\letsgojungle.exe X:\path\to\disk0\lgj_final
```

Two environment variables help during bring-up: `LINDBERGH_HLE_PERMISSIVE=1`
reports each unbound import once and carries on instead of stopping, so one run
enumerates the whole demand list; `LINDBERGH_HLE_TRACE=1` names every import as
it is entered.

### What the engine needed that was not obvious

**`XFree86-VidModeExtension`.** The game will not build its window without it,
and `libXxf86vm` is linked statically *into* the binary — so there was no
import to bind. The toolkit grew guest-function overrides for this: replace a
lifted function with a host body, by symbol name.

**`Display` is not opaque.** `struct _XDisplay` is public in `Xlib.h` and
`DefaultScreen(dpy)` is a macro that reads offset 132 directly.

**547 GL entry points, not 96.** `es::glh_helper::init_extensions` resolves
them through `glXGetProcAddressARB` and sets its feature flags only if they all
arrive. The 451 with no import get a synthetic address the runtime forwards to
the host driver.

**Cg, answered from the disc.** The game compiles its shaders at startup and
there is no Cg runtime here — but every shader ships twice, `vs.cg` beside
`vs.asm_gl`, the second being the first as `cgc` compiled it in 2005. Match the
source, return the compiled text.

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

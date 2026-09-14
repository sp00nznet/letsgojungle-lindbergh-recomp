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

## Status — it runs a full render loop, and presents a black frame

**Corrected.** An earlier version of this file claimed the attract mode
rendered. It does not. The call counts below are real and the render loop is
real, but the frame that reaches the screen is empty — measured, not assumed.

*Let's Go Jungle* boots from its own ELF, opens a window, loads its shaders and
runs a sustained render loop at ~32 fps:

```
[gl] 96 of 96 entry points bound
[crt] main at 0x08411ff0 (argc=1)
[pthread] created thread at 0x084f9d82, 1024 KB stack   ×3
[window] 1360x768
[glX] context created (pixel format 11)
[glX] current: NVIDIA GeForce RTX 5070/PCIe/SSE2 / 4.6.0 NVIDIA 595.97
[cg] indexed 197 precompiled shaders
```

| | per 60 s | per frame |
|---|---:|---:|
| `glXSwapBuffers` | 1,939 | — |
| `glVertex3f` | 127,080 | ~298 |
| `glBegin` / `glEnd` | 22,596 | ~53 |
| `glProgramEnvParameter4fvARB` | 81,029 | ~190 |
| `glBindTexture` | 106,550 | ~55 |
| `glProgramStringARB` | 189 | — |
| **non-black pixels presented** | **0** | **0** |

### What is ruled out

`LINDBERGH_FBSTATS=1` reads the back buffer with `glReadPixels` before each
swap and reports how much of it is lit. The instrument verifies itself: on one
frame it paints a colour nothing else would produce and reads it straight back,
which comes through at 100%. So the readback, the drawable and `GL_BACK` are
all correct, and the black frame is real.

With that, the following are measured and **not** the cause:

* **Shaders** — 189 ARB programs load, none rejected (`GL_PROGRAM_ERROR_POSITION_ARB` is −1 throughout).
* **Cg matching** — of the first 50 programs, all 50 matched on shader body *and* defines; none fell back to the looser key.
* **Render state** — colour mask `1111`, depth `GL_LEQUAL` cleared to 1.0, alpha test off, blend off, scissor off.
* **ARB programs enabled** — `glEnable(GL_VERTEX_PROGRAM_ARB)` and the fragment equivalent both reach the driver.
* **Framebuffer objects** — forcing every bind to the default framebuffer (`LINDBERGH_NO_FBO=1`) does not change it.

### Why it is black

The geometry is correct. `glVertex3f` submits (-1,1,0) (-1,-1,0) (1,1,0)
(1,-1,0) — a fullscreen quad already in clip space, which should pass straight
through. It does not appear, so the bound vertex program is applying a world
transform to coordinates that were already in normalised device space.

It is bound wrongly because **the disc ships one compiled form per shader and
the engine asks for several**. `shader/Cg/vs/vs.cg` has a single `vs.asm_gl`
beside it, but the engine compiles that shader repeatedly with different
`MODE_VS_1_1` / `MODE_VS_2_0` / `MODE_VP40` permutations — ~261 requests
against 197 precompiled pairs, with zero match failures, so variants collide
onto the same program. For a fullscreen blit pass the one it gets is a world
transform.

The matcher cannot separate them: across all 197 shaders there are only **two
distinct define sets**, because `cgc` records the whole block in every header
and it barely varies. Matching now identifies *which shader* correctly (a
shader's own code — everything after its last `#include` — is the tail of the
preprocessed source), but *which variant* is not recorded on the disc at all.

The fix is a real Cg compiler. `libCg.so` ships on the disc as a 32-bit x86 ELF
with 284 symbols — exactly what this toolkit lifts — and needs shared-object
loading in the runtime, which does not exist yet.

### Where the frame goes

The engine renders into framebuffer objects — 10 binds per frame — and the
frame ends with state setup and a swap, with no final draw to the default
framebuffer. At swap time the last render target set was 128×128, a small
offscreen buffer. The composite that should bring the scene to the screen never
lands, and why is not yet known.

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

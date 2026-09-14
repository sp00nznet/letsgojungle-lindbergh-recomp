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

## Status — attract mode runs

![Two giant spiders mid-leap on a jungle path, both players' rifles and
crosshairs on screen, combo counters running](docs/attract.png)

![A swarm of giant wasps in a cave, muzzle flash and tracer from the left-hand
rifle](docs/attract-cave.png)

The attract loop runs end to end: warning card, SEGA and CRIWARE logos, title,
the tutorial, a demo of the rail sequences, and the monthly ranking board.

*Let's Go Jungle* boots from its own ELF, opens a window, answers the cabinet's
base board, and runs its attract mode at 1360×768 with its own shaders,
textures and HUD. The frame above is `glReadPixels` on the back buffer just
before the swap, not a mock-up.

```
[gl] 96 of 96 entry points bound
[sega] 23 base board entry points answered
[game]  I/O Board 'SEGA ENTERPRISES,LTD.;I/O BD JVS;837-13551 ;Ver1.00;98/10'
[crt] main at 0x08411ff0 (argc=1)
[window] 1360x768
[glX] current: NVIDIA GeForce RTX 5070/PCIe/SSE2 / 4.6.0 NVIDIA 595.97
[cg] indexed 197 precompiled shaders
```

### What was actually wrong

An earlier version of this file said the black frame came from Cg shader
variants colliding onto one compiled program. That was wrong, and so were four
other diagnoses before it. The cause was one instruction.

`fxch st(2)` swaps the top of the x87 stack with the third register down.
Capstone spells it with **both** registers, the implicit `st(0)` first, so the
operand that matters is the second one — and the lifter was reading the first.
Every `fxch` in the binary became a swap of `st(0)` with itself: valid C, no
crash, no warning, and the x87 stack left in exactly the order the original
code used `fxch` to avoid. There are **906 of them** in this one game.

`_sShaderConstantTable::setMatrix44` moves three floats of every matrix column
through that stack. With the swap gone the three landed in each other's places,
so the world-view-projection matrix reached the vertex program with its first
column zeroed:

```
c[0] = (0        0         0         0)      <- should be (2/1360, 0, 0, 0)
c[1] = (0        0.002604  0         0)
c[2] = (0        0        -0.01005   0)
c[3] = (0.001471 0        -0.005025  1)      <- holding c[0].x
```

`clip.x` is then constant for every vertex in the scene: the whole world
collapses onto a single vertical line, one pixel wide, off to the side. The
engine submitted a complete frame — right geometry, right textures, right
shaders — into a degenerate projection. Fixed in
[pcrecomp](https://github.com/sp00nznet/pcrecomp), with a test that fails on
the old code.

### How it was found

Guessing had already produced five wrong answers, so the last stretch was
measurement only, each step narrowing by elimination:

| Question | Instrument | Answer |
|---|---|---|
| Does the engine draw at all? | per-target `glBegin` counts | 12,600 primitives, 70,816 vertices per frame |
| Does any of it rasterise? | every fragment program replaced with solid green | 100% of the screen |
| Is the texture black? | read the bound texture back at its own size | the 2048×2048 atlas is 100% lit |
| Are the coordinates zero? | fragment program emits `fragment.texcoord[0]` | non-zero and varying |
| So what does the shader do? | dump the program the Cg seam supplied | two instructions: `TEX`, then `MUL` by vertex colour |
| Then where does the colour go? | read the vertex program's constants at a real draw | `program.env[0]` is all zeros |
| Is the upload wrong? | print the bytes the guest hands to GL | the guest itself supplies the zeros |
| Is the matrix maths wrong? | both SSE multiplies reimplemented in C | byte-identical output — the multiplies are fine |

That left the transpose between the multiply and the upload, which is the x87
routine above. Several of those instruments were wrong before they were right
— the first framebuffer probe read a fixed 64×64 corner of an 800×600 target
and called a healthy buffer empty — so each one is now checked against a known
answer before its result is believed. They all live in the toolkit behind
`LINDBERGH_*` switches.

### The cabinet

A Lindbergh game does not talk to hardware directly; it calls SEGA's `amLib`,
which is statically linked into the binary. Every one of those calls failed
here, and the game stopped on **Error 11 — JVS I/O board is not connected to
main board** before it drew anything at all.

The game narrates all of this itself through `_sDebug::putConsole`, which goes
to a debug console the cabinet has and this does not. Binding it
(`LINDBERGH_CONSOLE=1`) is what turned the rest from guesswork into reading:

```
_sArcadeManager: amLibInit ret=-1
 SEGA BaseBD not available
_sInterfaceJvsManager: amJvsInit ret=-1
```

What answers now:

* **The base board** — `amLibInit`, `amLibIsBasebdAvailable`, `amJvsInit`, `amDongleInit`, `amDongleUpdate` and their predicates. `amJvsCheckInit` is a predicate, not a status code; answering it with the library's success value of 0 turned into "−5 JVS node(s) found".
* **A JVS I/O board**, at the transport. `amJvsSendRequest` and `amJvsRecvAcknowledge` are the only two functions replaced; the frames are real JVS (`E0 01 02 10 13` — sync, node, count, READ ID, checksum) so all 60 of the game's own packet builders and parsers run unmodified above them. It reports board identity, command and JVS revisions, and a feature list of two players, twelve buttons each, two coin slots and eight analog channels. Nothing is pressed, inserted or aimed: an attract mode wants a board that answers, not one that plays.
* **The battery-backed store**, at the four wrapper functions under the record layer, persisted to `lindbergh_nvram.bin`. Record layout, duplicate copies and CRCs are the game's own code and work untouched. Read takes the offset first and the buffer second; write takes them the other way round.

### What is assumed rather than emulated

The backup records are blank and the game cannot initialise them, because its
repair path writes through an EEPROM on an I2C bus that is not emulated. The
records then fail their CRC and the arcade framework latches **Error 15 — Game
Program Not Found** over everything.

A real cabinet ships with that store already written, so the runtime makes the
same statement: it suppresses that one error code and says so on the way past.

```
[sega] backup records are blank and cannot be initialised without the EEPROM
       bus; treating the bookkeeping as good
```

Every other error still reaches the game untouched. Emulating the EEPROM bus so
the game writes its own defaults is the next piece of work here.

### Not done

* No input — the JVS board reports no buttons, coins or gun position, so attract mode plays but nothing can be started.
* No sound.
* Cg is still answered from the disc's precompiled `.asm_gl` files rather than compiled. It has not mattered yet, but the variant ambiguity described below is real and will.

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

Then run it from the game tree, which is what `TEA_DIR` has to point at —
the cabinet's own launcher script sets it, and the engine will not find its
data without it:

```powershell
cd path	o\disk0
$env:TEA_DIR = (Get-Location).Path
$env:LINDBERGH_HLE_PERMISSIVE = 1
..uild\Release\letsgojungle.exe lgj_final
```

### Switches

Everything below is off unless set, and each one answers a specific question
rather than changing behaviour.

| Variable | What it does |
|---|---|
| `LINDBERGH_CONSOLE=1` | the engine's own `_sDebug::putConsole` messages — what it looked for and why it gave up |
| `LINDBERGH_FBSTATS=1` | how much of each frame is lit, and what the offscreen targets hold |
| `LINDBERGH_TRACE_FRAME=N` | every draw of frame N: bound programs, viewport, scissor, textures, and the target after it |
| `LINDBERGH_FP_SOLID=1\|2\|3` | replace every fragment program with flat green, a raw texture fetch, or the texture coordinates |
| `LINDBERGH_JVS=1` | the JVS frames in both directions |
| `LINDBERGH_NVLOG=1` | every backup store access |
| `LINDBERGH_SHOT=path` | write the back buffer to a BMP (`LINDBERGH_SHOT_FRAME` picks the frame) |
| `LINDBERGH_NVRAM=path` | where the battery-backed store lives (default `lindbergh_nvram.bin`) |

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

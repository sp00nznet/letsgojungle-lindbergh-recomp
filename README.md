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

## Status — it boots and reaches `main()`

`lgj_final` is a 12.7 MB `ET_EXEC` / `EM_386` binary that **ships its full
symbol table**, which is the single best thing about this target: 31,759
functions with names and sizes, so there is no function-discovery problem at
all. All of them lift, in 28 seconds, into 1,707,769 lines of C across 80
translation units — and that compiles and links into a 25 MB native
executable which then runs the game's entire C runtime:

```
> letsgojungle.exe lgj_final
[crt] __libc_csu_init at 0x0859fff8
[hle] glXGetProcAddressARB
[crt] main at 0x08411ff0 (argc=1)
...
[hle] XOpenDisplay
```

`_start`, every C++ static constructor in the binary — Xerces, Cg, the game's
own globals — then `main`. All of it executing lifted x86.

| | |
|---|---|
| Functions recovered | **31,759** — from the binary's own symbol table |
| Functions lifted | **31,759 / 31,759**, in 28 s |
| C emitted | 1,707,769 lines, 80 translation units |
| Instruction coverage | **99.94%** — 962 `/* TODO */ abort()` lines |
| Compiles and links | **Yes** — 25 MB executable |
| Boots | **Yes** — CRT, static constructors, `main` |
| Imports bound | 81 of 406 |

### What it asks for, in order

Run it with `LINDBERGH_HLE_PERMISSIVE=1` and every unbound import reports
itself once instead of aborting, so a single run enumerates the whole startup
path:

```
glXGetProcAddressARB            <- the only import the constructors need
pthread_mutex_lock / unlock
getcwd, realpath                <- works out where it is installed
pthread_mutexattr_*, pthread_mutex_init, pthread_cond_init
pthread_attr_*, sched_get_priority_max / min
pthread_create                  <- spawns a worker thread
pthread_cond_wait               <- and waits on it
XSetErrorHandler, XOpenDisplay  <- opens the display
```

So the next two jobs are **pthread** on Win32 threads, and then the window.

`XOpenDisplay` is the right place to stop. Reimplementing 50 Xlib calls on
Windows so they can hand a GLX context to WGL would be absurd — the seam gets
cut there instead, with a Win32 window and a WGL context behind an opaque
`Display *` the game never looks inside. That turns 68 X11 and GLX imports into
about a dozen shims.

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

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

### Not done

* No input — the JVS board reports no buttons, coins or gun position, so attract mode plays but nothing can be started.
* No sound.
* Cg is still answered from the disc's precompiled `.asm_gl` files rather than compiled. It has not mattered yet, but the variant ambiguity described below is real and will.

## Further reading

The detail moved to `docs/`, so this page stays a map.

| | |
|---|---|
| [docs/the-black-frame.md](docs/the-black-frame.md) | why the scene rendered nothing, and the five wrong answers before the right one |
| [docs/cabinet.md](docs/cabinet.md) | the base board, the JVS I/O board, and what is assumed rather than emulated |
| [docs/engine.md](docs/engine.md) | the five things this engine needed that a desktop does not have |
| [docs/switches.md](docs/switches.md) | the `LINDBERGH_*` runtime switches |
| [docs/disc.md](docs/disc.md) | what a clean game tree looks like |

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

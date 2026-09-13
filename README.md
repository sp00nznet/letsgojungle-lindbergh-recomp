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

## Status — blocked on the disc, not on the toolchain

Honest version, in two parts.

**The toolchain is ready.** The disc carves, the ELF pipeline lifts, the
runtime builds and runs 32-bit. This repo configures and passes its checks from
a fresh clone right now:

```
> cmake -S . -B build -A Win32
-- letsgojungle: no lifted code in .../generated yet, building the toolkit
   self-check only.
> .\build\lindberghrecomp\Release\lindbergh_rt_selftest.exe
ok: dispatch table, syscall layer, import names
```

**The disc is encrypted.** Carving the dump gets as far as named, correctly
sized payload files and no further:

```
> py -3.11 -m tools disc "Let's Go Jungle (World) (En,Ja) (Lindbergh Yellow) (Rev A).iso"
0x002b0000  sector 1376    SEGA_LINDBERGH   LETS_GO_JUNGLE               1.0 MB
0x003b8000  sector 1904    LINUX            CDROM                     1068.4 MB

disk0.img       1034084352      disk1.img          372736
disk9.img            372736     su1.dat          32891331
su2.dat              340450     frontend.set          878
```

Every one of those is ciphertext. `disk0.img` is the game's root filesystem and
the `main.elf` inside it is the thing this repo exists to recompile — under a
key that lives in the cabinet, not on the disc.
[The measurements are written up here](https://github.com/sp00nznet/lindberghrecomp/blob/master/docs/disc-format.md),
including the evidence that this disc shares its key with *Initial D 4*.

**lindberghrecomp does not decrypt Lindbergh media and neither does this repo.**
What unblocks it is a `disk0.img` you can mount or an install off a board's own
hard disc. Everything downstream is already waiting for it.

## What happens when the ELF turns up

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

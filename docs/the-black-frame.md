# The black frame

The scene rendered nothing for a long time, and the
reason was not in this game at all.

## What was actually wrong

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

## How it was found

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

# What the engine needed

Five things this game wanted that a desktop does
not have, and how each is answered.

## The five things the engine needed

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

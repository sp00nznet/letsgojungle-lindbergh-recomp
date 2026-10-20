# Roadmap

## Next

- **Input.** The JVS board answers but reports no buttons, coins or gun
  position, so nothing can be started. The toolkit's input queue already works
  for the GLUT titles; this game reads it over GLX.
- **Sound.** It waits on `libsegaapi` in the toolkit.
- **A one-click `Setup.cmd`** that checks the prerequisites, asks for the game
  tree, lifts, builds, and leaves a launcher. It also puts the game on
  recomp-launcher.
- **Headless recording** (`--headless --record out.mp4`), once the toolkit has
  it.

## Deferred

- Compiling Cg at runtime instead of matching the disc's precompiled `.asm_gl`
  files. It's only needed once the variant ambiguity bites.
- Two-player light guns on real hardware.

## Out of scope

- Decrypting the disc, or distributing any game data or lifted source.

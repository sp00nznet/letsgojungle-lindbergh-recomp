# Switches

All off unless set. Each answers a specific question rather
than changing behaviour.

## Switches

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

# Changelog

All notable changes to this project. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and versions follow
[SemVer](https://semver.org/). Nothing has been tagged yet. Everything so far
is under Unreleased, taken from the commits.

## [Unreleased]

### Added
- The host (`games/letsgojungle/src/host.c`) and the build: lift the ELF, link
  it against lindberghrecomp, run it from the game tree.
- The game boots through its CRT and static constructors to `main`, opens a
  1360×768 window with a real GL context, and loads its shaders and textures.
- Attract mode renders end to end at ~32 fps: warning card, logos, title,
  tutorial, a demo of the rail sequences, and the ranking board.
- Docs on the black frame, the cabinet, the engine, the switches and the disc.
- `CHANGELOG.md`, `ROADMAP.md`, `CONTRIBUTING.md`, `CONTRIBUTORS.md`, and CI
  that builds and runs the toolkit's self-check on every push and PR.

### Changed
- The toolkit is pinned to lindberghrecomp `d65e3b4`. That brings its Cg
  shader table of 2048 (lindberghrecomp#1) and self-check headers that build
  from a fresh clone (lindberghrecomp#2).
- The README follows the house layout: a Getting Started from a clean machine,
  Usage, Building from source, License and Contributors.

### Fixed
- `.gitignore` catches the game ELF, which has no extension (`lgj_final`).
- The README's run commands, whose backslashes had become control characters.
- `docs/disc.md` links the toolkit's `main` branch, not `master`.

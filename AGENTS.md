# AGENTS.md

This is a fork of [mGBA](https://github.com/mgba-emu/mgba) (Game Boy Advance / GB / GBC emulator), MPL 2.0. Commit history tracks upstream closely; see `CONTRIBUTING.md` and `PORTING.md` for authoritative conventions. **Upstream does not accept AI-generated code in PRs** — keep that in mind before opening upstream contributions.

## Build

CMake (≥3.12), C11 + C++ (Qt port only). GCC, Clang, MSVC all supported.

```bash
# CLion uses cmake-build-debug/; CMakePresets provide clang + gcc (Ninja, Debug, export compile_commands.json)
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug
cmake --build build
```

- All optional dependencies are auto-detected and silently disabled if missing. The configure summary (printed at the end) lists what's on/off — read it.
- **Windows requires libepoxy** unless `LIBMGBA_ONLY` or `SKIP_LIBRARY` is set (`CMakeLists.txt:750` hard-fails otherwise).
- Windows dev builds: MSYS2 is the recommended path (see `README.md`); Visual Studio needs vcpkg.
- macOS: use homebrew for deps, and do **not** run `make install` (breaks the bundle layout).
- `version.c` and `include/mgba/flags.h` are **generated** into the build dir from `src/core/version.c.in` / `flags.h.in` via `version.cmake` (which calls `git describe`). Don't edit the generated files.

## Testing

Testing is opt-in via CMake flags; nothing runs by default.

- `BUILD_SUITE=ON` — cmocka unit tests, run with `ctest`. Required dep: cmocka (auto-disabled if absent).
- `BUILD_TEST=ON` — fuzzers (`mgba-fuzz`, `tbl-fuzz`).
- `BUILD_CINEMA=ON` — video regression suite (needs ROMs under `cinema/gb` and `cinema/gba`).
- `BUILD_PERF=ON` — profiling tool.
- Qt tests (`src/platform/qt/CMakeLists.txt:597`) need QtTest; Python tests (`src/platform/python/CMakeLists.txt:57`) need `BUILD_PYTHON` and run via `setup.py ... pytest`.

**C test naming** (`src/platform/test/CMakeLists.txt:29`): a source at `src/<dir>/test/<name>.c` becomes the ctest label `<dir>-<name>` (the `-test` path segment is stripped). Run one test:

```bash
ctest --test-dir build -R '^util-circle-buffer$'      # matches src/util/test/circle-buffer.c
ctest --test-dir build -R '^gba-test-core$'           # matches src/gba/test/core.c
ctest --test-dir build -R '^platform-qt-'             # Qt tests
```

## Coding style (enforced — see CONTRIBUTING.md)

`.clang-format` is present (WebKit base, tabs, 120 cols). Additional rules not fully captured by the formatter:

- C structs: Capitalized, **not** `typedef`'d; associated functions start with the struct name (e.g. `LocalStructCreate`).
- File-soped C statics and static functions: prefix with `_`.
- C++ classes live in namespaces; Qt port uses namespace `QGBA`. Members: `m_`, static members: `s_`.
- Braces on the same line; **braces required even for single-line `if`/`else`**.
- C: use `0` (not `NULL`), prefer `bool`/`true`/`false`. C++: use `nullptr`.
- Header guards: `FILE_NAME_H` (C), `QGBA_FILE_NAME` (Qt, legacy).
- Commit messages start with the component name (e.g. `GBA Memory:`, `Qt:`, `Util:`, `Core:`). See `CONTRIBUTING.md` for the component list.

## Layout

- `src/arm`, `src/sm83` — CPU cores (ARM = GBA, SM83 = GB family).
- `src/gba`, `src/gb` — console-specific emulation.
- `src/core` — shared core infrastructure.
- `src/util` — common utilities (VFS, hashing, image, patches, GUI primitives).
- `src/debugger` — CLI + GDB stub debugger.
- `src/feature` — optional features (ffmpeg, sqlite3, updater).
- `src/script` — Lua scripting engine.
- `src/platform/<port>` — frontends: `qt`, `sdl`, `libretro`, `python`, `3ds`, `switch`, `wii`, `psp2`, `windows`, `posix`, `opengl`, `test`, `example`, `headless-main.c`.
- `src/platform/cmake` — CMake helper modules (`FindFeature`, `FindFunction`, `DebugStrip`, `ExportDirectory`).
- `src/third-party` — bundled deps: zlib, libpng, sqlite3, inih, lzma, discord-rpc.
- `include/mgba`, `include/mgba-util` — public headers.
- `cinema/` — fixtures for the video test suite (gitignored ROMs).

Notable CMake options: `M_CORE_GBA` / `M_CORE_GB` (toggle cores), `BUILD_QT` / `BUILD_SDL` / `BUILD_LIBRETRO` / `BUILD_PYTHON`, `DISABLE_DEPS` (strips most features), `LIBMGBA_ONLY` (library-only static build), `USE_LUA=JIT|<ver>`, `SDL_VERSION=1.2|2|3` (default 3, falls back).

## Ports

In-progress ports live on `port/<name>` branches and merge into `port/crucible` (see `PORTING.md`). Platform-specific code goes under `src/platform/<name>`; keep changes there and avoid invasive edits to the shared tree.

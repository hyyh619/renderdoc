# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

RenderDoc is a frame-capture based graphics debugger for Vulkan, D3D11, D3D12, OpenGL, and OpenGL ES on Windows, Linux, Android, and Nintendo Switch. This is a VeriSilicon fork tracking the upstream `v1.x` branch; local changes so far are build-environment tweaks (VS2022 upgrade, warnings-as-errors disabled) and small code-style/debug-info adjustments rather than a divergent product.

## Building

The `build/` directory here is already configured for a **macOS Debug** build with clang, `ENABLE_VULKAN=ON` and `ENABLE_GL=ON` (D3D/Metal/GLES off). To (re)build:

```
cmake -DCMAKE_BUILD_TYPE=Debug -Bbuild -H.   # configure (already done)
make -C build                                # build; add -jN for parallelism
```

Outputs land in `build/bin` and `build/lib`. `build/compile_commands.json` exists for clangd/tooling.

- **Windows**: open `renderdoc.sln` (upgraded to VS2022 in this fork). `Development` config for day-to-day work, `Release` for shipping/perf. No external deps beyond the Windows SDK.
- **Toggle drivers/features** at configure time, e.g. `cmake -DENABLE_GL=OFF ...`. Override compilers with `CC`/`CXX`.
- **Android**: separate build tree with `-DBUILD_ANDROID=On -DANDROID_ABI=armeabi-v7a`.

See [docs/CONTRIBUTING/Compiling.md](docs/CONTRIBUTING/Compiling.md) and [Dependencies.md](docs/CONTRIBUTING/Dependencies.md) for platform details.

## Testing

Tests live in [util/test/](util/test/) and are driven by a Python harness that loads the built RenderDoc module. First build the `demos` program (`cmake -Bbuild -Hdemos && make -C build`), then:

```
cd util/test
python3 run_tests.py --renderdoc /path/to/build/lib --pyrenderdoc /path/to/build/lib --demos-binary /path/to/demos_x64
```

- `-l` lists tests; `-t <regex>` includes, `-x <regex>` excludes specific tests.
- `--in-process` runs tests in one Python process (each test is normally a child process so a crash doesn't abort the run) — useful for debugging.
- `--slow-tests` includes long-running tests (excluded by default).
- The Python version **must match** the one RenderDoc was built against. `tmp/` and `artifacts/` are wiped on each run; the log ends up in `artifacts/`.

Unit tests for core data structures live alongside the code (e.g. `renderdoc/core/intervals_tests.cpp`, `renderdoc/replay/basic_types_tests.cpp`).

## Architecture

Three layers, built roughly bottom-up:

- **`renderdoc/`** — the core capture/replay library (produces `librenderdoc`). It has no UI dependency and is the heart of the tool.
  - `core/` — capture orchestration, the replay proxy (`replay_proxy.*`), resource management (`resource_manager.*`), remote server/target-control for cross-machine and Android capture.
  - `driver/` — per-API back-ends (`vulkan/`, `gl/`, `d3d11/`, `d3d12/`, `metal/`, plus `dxgi/`, `dx/`, `ihv/` for vendor extensions). Each driver both **hooks** the live API to serialise a capture and **replays** the serialised commands. Drivers can be compiled in/out independently.
  - `replay/` — the API-agnostic replay controller (`replay_controller.*`), replay/replay-driver interfaces (`replay_driver.*`, `dummy_driver.*`), and `entry_points.cpp` (the public C API surface).
  - `serialise/` — the serialisation framework underpinning capture files (structured data model).
  - `api/replay/` — **the public interface headers** (`renderdoc_replay.h`, pipeline-state structs, `rdcarray.h`/`rdcstr.h`, enums). These are the contract shared with the UI and Python bindings; changes here ripple widely.
  - `common/`, `os/`, `maths/`, `shaders/`, `strings/` — shared support code, OS abstraction, math, embedded shaders.
  - `hooks/` — function-hooking machinery for intercepting API calls.

- **`qrenderdoc/`** — the Qt UI built on top of the core library.
  - `Code/Interface/` — the `QRDInterface` layer that adapts the core API for the UI and Python (`PersistantConfig`, `RemoteHost`, extensions).
  - `Code/pyrenderdoc/` — Python bindings (PySide2/`swig`-style bridge); `ENABLE_PYRENDERDOC` builds the `pymodules`.
  - `Windows/`, `Widgets/`, `Styles/` — the actual panels (texture viewer, mesh viewer, pipeline state, etc.).

- **`renderdoccmd/`** — small CLI utility for tasks like launching captures, running the replay host, and Android support. `renderdocshim/` is a tiny kernel32-only Windows DLL for global hooking.

The **capture → replay** flow is the central mental model: a driver serialises live API calls into a capture file (via `serialise/`), then on replay the same driver reconstructs GPU state through the `ReplayDriver` interface, and the `ReplayController` / `ReplayProxy` expose it to the UI or Python — possibly across a network/Android boundary via `remote_server`.

## Coding style

`clang-format` (`.clang-format` at root; `util/clang_format_all.sh` formats everything) handles braces/indent/spacing. Beyond that, **match surrounding code**, and note these project-specific rules (from [docs/CONTRIBUTING/Developing-Change.md](docs/CONTRIBUTING/Developing-Change.md)):

- Use `NULL`, not `nullptr`.
- `auto` only for prohibitively-long types (STL iterators, lambdas); everywhere else spell out the type.
- Use `rdcarray` instead of `std::vector` and `rdcstr` instead of `std::string`. Keep STL to a minimum (`std::map`/`set`/`function`, `std::sort`/`lower_bound`, type-traits are fine).
- All strings are UTF-8 (`char*`); wide strings only for Windows OS interop. Never assume ASCII.
- No Hungarian notation except `m_` for members (existing `p`-for-pointer prefixes are being phased out; keep them only when mirroring hooked-function parameter names).
- If any branch of an `if`/`else` needs braces, brace all branches.

Keep PRs reasonably sized (upstream guideline: under ~1000 changed lines) and split large features into incremental, per-driver landings where possible.

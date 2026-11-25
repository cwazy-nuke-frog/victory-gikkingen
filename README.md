# VIctory fork: GTA VI on Linux

Fork of [fxceless/victory-fork](https://github.com/fxceless/victory-fork), which is a fork of
[VIctory](https://github.com/Marvisak/VIctory), an Xbox GDK compatibility layer.
Target: the Xbox Series build of GTA VI on Linux, through Proton.
This repository contains no game files, code or assets. You need your own copy of the game.
It is not affiliated with or endorsed by the game's developer or publisher.

## Status (2026-11-25)

- Current state: close-to-console rendering. Work in progress.
- Scene: prologue, prison. Cell, characters, clothing and yard are drawn.
- Defects: light-blue blocky tiles at the bottom edge in some launches. Lighting changes between
  launches. Approximately 1 launch in 3 stops at init (exit code 3). Launch again.
- Rate: approximately 3 fps (300 ms per frame). Main limit: waits on GPU fences during copy-back.

## Demo

https://github.com/user-attachments/assets/193b79bb-afd9-42af-8c03-873cdf0934ce

Prologue, prison dormitory, live play on this branch. 1080p, 102 s. The desktop outside the game
window is blurred. Media are kept out of this repository (asset policy, rule #1).

## Changes on this branch

Base: fxceless/victory-fork `c96b4f2`. Additions:

- Aliased title pages: a write through one mapping reaches all mappings (port of upstream
  VIctory #1122). Result: the cell wall is drawn in each launch.
- DirectStorage page-destination writes notify the GPU side (port of fork PR #571).
- Packed textures: a newer view makes an older view stale only at the same start address. A
  texture on the card is refreshed when the card is full (port of fork `310c6eb6`, two parts).
  Result: correct clothing colours.
- Compute-only shader loop limit, `VICTORY_SHADER_LOOP_LIMIT_CS` (port of Danigamer05 `3de85981`).
- Wave64 programs on 32-lane hosts: shared-memory ordering rule for the two wave halves.
- Merged community PRs: fork #24, #92, #95, #108, #127, #156, #213, #218, #257, #259, #265;
  upstream #454; ray-tracing traces branch by sedzz (off by default).
- `VICTORY_PIPELINE_WORKERS`: pipeline worker count. `VICTORY_VIEW_HEAP_FIT=1`: the title's view
  heap fits under the 1,000,000 views of NVIDIA and vkd3d-proton.

Measurements and causes: refer to [port/FINDINGS.md](port/FINDINGS.md).

## Run configuration

- Build: `cargo xwin build --profile diag -p victory --features diagnostics
  --target x86_64-pc-windows-msvc`. Run `victory.exe` under Proton Experimental.
- Switches used for the demo: `VICTORY_VIEW_HEAP_FIT=1 VICTORY_PIPELINE_WORKERS=10
  VICTORY_LEAN_WINDOWS=1`, `VKD3D_CONFIG=descriptor_heap`,
  `VKD3D_DISABLE_EXTENSIONS=VK_NV_raw_access_chains`.

## Test configuration

- CPU: AMD Ryzen 5 5600X, 6 cores, 12 threads. RAM: 32 GB.
- Game GPU: NVIDIA RTX 3090, 24 GB, driver 620.70.06, PCIe 4.0 x8, Resizable BAR on.
- Display GPU: AMD RX 5700 XT.
- OS: Fedora Linux 44, kernel 7.3.3.
- Proton Experimental 11.0 (2026-11-12). vkd3d-proton git build `2576c0b`.

## Method

Agentic loop. A mini swarm of 6 or 7 agents works on this repository in a continuous loop.
Tasks: monitor VIctory and its forks, port fixes, run unattended trials, evaluate the results.
Loop tooling in [port/](port/): trial queue, sandboxed launch, image metrics, paired statistics,
security gate. Refer to the header of each script in `port/run/`. License of `port/`: MIT.

## Credits

- [VIctory](https://github.com/Marvisak/VIctory) by Marvisak and its contributors: the base of
  everything here. License GPL-3.0-or-later; this fork uses the same license.
- [fxceless/victory-fork](https://github.com/fxceless/victory-fork) and its contributors: the
  base line of this branch.
- Authors of the ported and merged commits, credited in the history of this branch.

# VIctory

Runs the title (Xbox Series X|S) natively on Windows, from files
the user supplies.

The title's x86-64 code runs unchanged inside our process; VIctory supplies
the platform underneath it and converts the title's data, ahead of time where
it can and on first use where it must (shaders). See
[docs/architecture.md](docs/architecture.md).

## Status (2026-11-23)

**Not playable yet.** The title boots, runs its menus at its own 30 fps and
loads the world, where rendering fails.

| Stage | State |
|-------|-------|
| Boot, first-boot screens (accessibility, language, settings, legal) | works, 30 fps |
| Intro video with sound | works |
| Start screen, profile and saves | works |
| Loading into the world (about 50 s) | works |
| The world (first mission's intro) | reached and runs with indirect draws; its title card renders, the scene after it is still dark; 0.5-1.8 s per frame |

Tested on one machine: Windows 11, AMD RX 6700 XT.

> **Warning.** Entering the world on a real GPU can currently hang it, and
> Windows then resets the display driver (every monitor goes black for a
> moment). To look further, use a diagnostics build with
> `VICTORY_D3D12_WARP=1`, which renders on the software adapter.

What each part does today and what it still needs:

| Subsystem | Today | Needs |
|-----------|-------|-------|
| Loader | Runs the title's executable and its DLLs in our process; crash reports name the faulting code and are kept in `crashes.log` | – |
| Runtime services | One signed-in offline user, licences, packages, saves that survive a kill at any step, text entry dialog | Speech is silent; the dialog and relaunch are untested live |
| Network | Offline by design: the title cannot reach the network | – |
| Files and streaming | Archives and streamed data read, zlib and BCPack decoded in parallel | Loading is limited by writes into title memory |
| Input | XInput pads, keyboard, keyboard stand-in pad | Trigger rumble |
| Audio | Menu, video and loading sound; XMA voices and Opus streams decode | Reverb is silent; Opus results unverified against the title |
| GPU: menus | Draws, compute and presentation on host D3D12; only changed title memory is copied | Frame pacing drifts on one screen in release builds |
| GPU: world | Mesh pipelines, shader-written buffers, descriptor lifetimes, indirect execution (arguments clamped), pipelines made by worker threads and kept on disk | Most of the scene is still missing (untranslated shaders drop draws); frames take seconds; real queries, depth bounds, HDR and scaling are missing |
| Shaders | Console GPU code recompiled to DXIL at first use, cached on disk. Menus: all. World: mesh 8,191 of 8,506, vertex/pixel 10,617 of 12,346, compute 1,643 of 2,539 | 64-bit image atomics, merged geometry stages, the ray-tracing builder's shaders; some compute programs need a GPU with 64-lane waves |
| Textures | Console tiled layouts and detiling for the modes the title uses | Render-target metadata planes |
| Series S mode | Runs through the menus to the loading screen | The world, as above |

## Getting started

Requirements: Windows 10/11 x64, a CPU with AVX2, a Direct3D 12 GPU with
shader model 6.6 (and mesh shaders for the world), Rust (pinned in
`rust-toolchain.toml`).

The title also needs several GiB of available Windows commit at startup.
Check **Committed** under **Task Manager → Performance → Memory**. If
`crashes.log` reports Windows error 1455, close other applications or increase
the Windows paging file size, then retry.

```powershell
cargo xtask install-hooks                # asset-policy check on every commit
copy victory.example.toml victory.toml   # then set your paths (outside the repo)
cargo test --workspace                   # needs no game data
cargo run -p victory -- config           # check your paths
```

Run the title:

```powershell
cargo build --release -p victory
target\release\victory.exe run <game dir>\<the title's exe> --execute
```

The window caption shows the framerate. Saves, settings and `crashes.log`
are kept under the data folder (default `%LOCALAPPDATA%\victory\data`), derived
data under the cache folder.

Input: XInput pads. With none connected the keyboard is a pad while the
title's window has focus:

| Keys | Pad |
|------|-----|
| W A S D / I J K L | left / right stick |
| arrows | D-pad |
| Enter, Backspace, X, Y | A, B, X, Y |
| Q / E, Z / C | shoulders, triggers |
| F / H | stick clicks |
| Esc, Tab | Menu, View |

## Commands

```
victory inspect <exe|dll> [--imports] [--exports]
victory census [paths...] [--missing] [--strict]
victory lib-exports <import.lib> [--list]
victory shader-info [paths...] [--verbose] [--disasm]
victory translate [paths...] [--validate] [--pipelines] [--list] [--jobs N]
victory run <exe> [--execute] [--unimplemented trap|zero] [--strict]
            [--console series-x|series-s] [-- <title arguments>]
victory config
```

Logging: `VICTORY_LOG=victory_loader=trace,info` (a release build logs
info and above).

## Debugging

A release build has no debugging aids. Build them in with

```powershell
cargo build --profile diag -p victory --features diagnostics   # target\diag\victory.exe
```

| Variable | Effect |
|----------|--------|
| `VICTORY_LOG=<target>=debug,...` | debug and trace output |
| `VICTORY_STACKS=20,60` | every thread's stack at those seconds |
| `VICTORY_WATCH=<rva>,...` | logs a title u32 when it changes |
| `VICTORY_KEYS=30:Enter,35:X` | presses keys at those seconds, focus or not |
| `VICTORY_CRASH_SAVE=<rva>:<len>,...` | saves title memory to the cache folder on a crash |
| `VICTORY_CAPTURE=<dir>` | saves presented frames (powers of two, or `VICTORY_CAPTURE_FRAMES=<list>`) |
| `VICTORY_DUMP_TARGETS=1` | with the two above, saves every render target of those frames |
| `VICTORY_D3D12_DEBUG=1` | host debug layer |
| `VICTORY_D3D12_GBV=1` | with the above, GPU-based validation (bad descriptor use reported, not faulted; slow) |
| `VICTORY_D3D12_DRED=1` | on a GPU failure, the allocation hit and the commands in flight |
| `VICTORY_PIPELINE_WAIT_MS=24` | reduce the per-frame wait for background pipeline creation from the default of 500 ms. A draw can be omitted until its pipeline is ready |
| `VICTORY_SKIP_PIPELINES=<hash>,...` | omit draws and dispatches, including indirect ones, whose pipeline contains a listed 16-digit shader hash |
| `VICTORY_GPU_TIMING=1` | with the above, each list of a submission run and timed on its own; slow lists logged with their slowest commands |
| `VICTORY_D3D12_WARP=1` | host device on the software adapter (slow; cannot reset the driver) |
| `VICTORY_SHADER_LOOP_LIMIT=<n>` | translated shader loops stop after `n` iterations per lane instead of 2^20 (own shader cache entries); if a lower `n` makes world frames much faster, loops hitting the limit are what makes them slow |
| `VICTORY_SHADER_LOOP_LIMIT_CS=<n>` | the same limit for compute programs only (unset: they follow `VICTORY_SHADER_LOOP_LIMIT`). A lane that reaches the limit leaves without its writes, so a low general limit also breaks genuine compute loops: with 32 the hair simulation's per-vertex constraint pass never stores and strands scatter around heads. `VICTORY_SHADER_LOOP_LIMIT=32 VICTORY_SHADER_LOOP_LIMIT_CS=128` keeps the low limit for the other stages (on an RTX 2080 SUPER 1024 also kept the hair in place but showed a smeared copy of the character along the bottom of the screen; 128 did not) |
| `VICTORY_AUDIO_CAPTURE=<file.wav>` | records what the title plays (our process only) |
| `VICTORY_XMA_DUMP=<dir>` | saves the first 32 XMA voices' context and input |

Every build appends crashes and host GPU failures to `crashes.log` in the
title's data folder.

## License

GPL-3.0-or-later. See [LICENSE](LICENSE).

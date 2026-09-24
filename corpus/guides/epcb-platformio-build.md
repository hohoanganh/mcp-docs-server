---
id: epcb-platformio-build
title: "EPCB: Build and flash with PlatformIO (not the original Makefile)"
section: guide
tags: epcb, platformio, build, flash, upload, release, makefile, workflow, stm32l151, bsf, swd, stlink
summary: EPCB's ak-base-kit-pio fork builds with PlatformIO, not the original application/Makefile - use `pio run -e app`, set RELEASE via platformio.ini build_flags, set the version only in APP_VERSION, and seed BSF only on boards still running bootloader < 0.0.2.
---

# EPCB: Build and flash with PlatformIO

> **Scope.** This guide describes the EPCB fork
> [`hohoanganh/ak-base-kit-pio`](https://github.com/hohoanganh/ak-base-kit-pio),
> which ports the AK base kit to PlatformIO. The AK **kernel** API (task,
> message, timer, fsm/tsm) is unchanged - all other guides in this knowledge
> base apply as-is. What differs is the **build system, flashing and release
> flow**. Where this guide and a Makefile-based guide disagree, this one wins
> for EPCB projects.

## Do not look for `application/Makefile`

The upstream AK base kit builds with `application/Makefile` and `boot/Makefile`.
**Those files do not exist in the EPCB fork.** Any advice to edit
`RELEASE_OPTION` in `application/Makefile` (see the `agent-workflow` guide) does
not apply here - there is no such file to edit.

The leftover `Makefile.mk` files under `sources/` are kept for reference only;
PlatformIO never reads them.

| Upstream (Makefile) | EPCB fork (PlatformIO) |
|---|---|
| `application/Makefile` | `platformio.ini`, section `[env:app]` |
| `boot/Makefile` | `platformio.ini`, section `[env:boot]` |
| `RELEASE_OPTION = -URELEASE` | `-DRELEASE` in `build_flags` (see below) |
| `make` | `pio run -e app` |
| `make flash` | `pio run -e app -t upload` |

## Build and flash

```bash
pio run -e app                 # build application
pio run -e boot                # build bootloader
pio run -e app  -t upload      # flash app  (ST-Link / SWD)
pio run -e boot -t upload      # flash boot (blank board needs both)
pio run -e app  -t bsf         # seed BSF - only for bootloader < 0.0.2, SEE BELOW
pio run -t clean
```

A blank board needs boot, then app. With bootloader **0.0.2 or later** (source
base v1.1.0+) that is all - the bootloader repairs an empty BSF by itself. Only
boards still running an older bootloader also need `bsf`.

Each env has its own flash limit (`board_upload.maximum_size`: app 116K, boot
8K), so PlatformIO reports real usage and fails the build if an image outgrows
its region. The bootloader already uses ~83% of its 8K.

## RELEASE flag

`RELEASE` is a compile-time define in `platformio.ini`, not a Makefile variable.

- `[env:boot]` ships `-DRELEASE` (release build).
- `[env:app]` - comment the `-DRELEASE` line out while developing so
  `APP_DBG`/assert output stays on, and put it back when shipping.

The debug-vs-release reasoning in the `agent-workflow` guide still holds; only
the place you flip the switch changes.

## BSF and the bootloader version

Check the bootloader version first - it prints `[BOOT] version: x.y.z` on the
console at reset.

**Bootloader 0.0.2 or later (source base v1.1.0+):** no seeding needed. The
bootloader validates the application's own vector table (initial SP inside
SRAM, Thumb reset vector inside the app region). If BSF is empty or was erased
mid-write but the app image is valid, it repairs the broken BSF fields and runs
the app (`[BOOT] share boot repaired`). If the app region is empty it waits for
a UART upload instead of jumping into garbage.

**Bootloader 0.0.1:** after flashing `app` over SWD the board only blinks an LED
and never runs the application. It looks bricked. It is not. That bootloader
jumps to the application only when the *boot share flash* (BSF) at
`0x08002000` contains both:

- `fw_app_cmd.cmd == SYS_BOOT_CMD_NONE`, and
- `current_fw_app_header.psk == FIRMWARE_PSK`

BSF is normally written by the UART bootloader / OTA update path. Flashing
directly with ST-Link never touches it, so on a fresh chip BSF is still erased
- which on STM32L1 reads as `0x00`, not `0xFF` - and the bootloader falls into
its "unexpected status" branch: `while(1)` with a blinking LED. Fix once with:

```bash
pio run -e app -t bsf
```

Better: flash the current bootloader from the source base
(`release/boot/ak_base_kit_boot_v<x.y.z>.bin` at `0x08000000`). See
`docs/known-bugs.md` #3 in the source base for the full failure analysis.

## Memory map

```
0x08000000  boot   8K    sources/boot/platform/stm32l/ak.ld
0x08002000  BSF    4K    boot share data flash (seeded by -t bsf)
0x08003000  app  116K    sources/application/platform/stm32l/ak.ld
```

## Never remove the linker page-size flags

`pio_build_flags.py` appends these to `LINKFLAGS`, and they are **mandatory**:

```
-Wl,-z,max-page-size=4  -Wl,--nmagic
```

Why: `ld` aligns PT_LOAD segments to a 64K page by default. The app segment
lives at `0x08003000`, so default alignment drags its `p_paddr` back to
`0x08000000` and pads ~12K of junk in front. `pio run -t upload` flashes via
openocd `program firmware.elf`, and **openocd writes program headers, not
sections** - so it would write that junk over the bootloader at `0x08000000`
and wipe BSF. The board dies immediately after a successful-looking flash.
These two flags force the segment to start exactly at `0x08003000`.

If a build ever reports the app segment starting below `0x08003000`, stop and
check these flags before flashing.

## Build output location

`build_dir` points at `%TEMP%/pio_build_ak_base_kit`, not `.pio/` in the project.
The repo lives in a OneDrive-synced folder, and OneDrive locks `.o` files
mid-build (`ar.exe: unable to rename ...: Permission denied`).

On Linux/macOS `%TEMP%` is unset, which resolves to `/pio_build_ak_base_kit` at
the filesystem root and fails with `PermissionError`. Export a writable `TEMP`
first:

```bash
export TEMP="$HOME/.cache"
```

Release artifacts are copied to `release/{app,boot,bsf}/` by
`pio_copy_release.py`, named with `APP_VERSION`. Bump `-DAPP_VERSION` in
`platformio.ini` before a release build.

## Version: APP_VERSION is the only source

Since source base v1.1.2, `-DAPP_VERSION="x.y.z"` drives both the release file
name and the version the firmware reports (`App version: x.y.z.0` at boot and
from the shell `ver` command): `pio_build_flags.py` splits it into
`APP_VER_MAJOR/MINOR/PATCH` and `app.h` builds `APP_VER` from them. Do not edit
`APP_VER` by hand. (Before v1.1.2 it was hard-coded to `0.0.0.3`.)

## Compile flags from extra scripts must reach `projenv`

`pio_build_flags.py` is a POST extra script. By the time it runs, PlatformIO has
already cloned `projenv` - the environment that compiles files in `src_dir` -
from `env`. Appending `CFLAGS`/`CXXFLAGS`/`CPPDEFINES` to `env` alone reaches no
project source (link flags still work, the link step uses `env`). Apply compile
flags to both: `Import("env", "projenv")`. Before source base v1.1.2 this was
wrong and `-std=gnu99 / -std=gnu++11` were never applied; verify with
`pio run -v`.

---
id: epcb-start-project
title: "EPCB: Start a new product from ak-base-kit-pio"
section: guide
tags: epcb, start, new, project, bootstrap, scaffold, source-base, template, platformio, version, git, release
summary: EPCB products start from the ak-base-kit-pio source base (PlatformIO), not from an upstream base-kit tarball - clone a tagged version, drop the source-base history, and record which base version the product came from.
---

# EPCB: Start a new product from ak-base-kit-pio

> **Scope.** Use this instead of the `start-project` guide (and instead of the
> `start_ak_project` tool) for EPCB products. That flow downloads an upstream
> `ak-base-kit-stm32l151` tarball, which builds with Makefile and lacks EPCB's
> PlatformIO setup, BSF tooling and release scripts.

## Source base

[`hohoanganh/ak-base-kit-pio`](https://github.com/hohoanganh/ak-base-kit-pio) -
AK base kit ported to PlatformIO, target STM32L151CBT6. Used by IPMS, OPMS,
Smart PDU, EACC and other EPCB products.

It is a **template you copy**, not a shared library you link against. Firmware
needs to stay reproducible for years: a shipped product must rebuild to the same
binary long after the source base has moved on. Pointing several products at one
live copy means fixing the kernel for product A silently changes B and C.

## Create the product repo

Clone the **latest tag** (see `CHANGELOG.md` in the source base; `v1.1.2` at the
time of writing), then drop the source-base history so the product starts its
own:

```bash
git clone --depth 1 --branch v1.1.2 \
  https://github.com/hohoanganh/ak-base-kit-pio.git my-product
cd my-product
rm -rf .git
git init
```

Then, in the product's `README.md`, record the base version:

> Khởi tạo từ ak-base-kit-pio **v1.1.2**

This single line is what makes it possible later to tell which products are
missing a given source-base fix. Without it there is no way back.

Do not work out of a OneDrive-synced folder: OneDrive locks files inside `.git`
and mid-build `.o` files. Keep repos under `C:\Work\` or `D:\dev\`.

## First build

```bash
pio run -e boot -t upload      # 1. bootloader
pio run -e app  -t upload      # 2. application
```

With the bootloader shipped since source base v1.1.0 (bootloader 0.0.2+) that
is enough - it repairs an empty BSF by itself. Only a board still carrying
bootloader 0.0.1 also needs `pio run -e app -t bsf`. See the
`epcb-platformio-build` guide.

Before writing product code, read `docs/known-bugs.md` in the source base: it
lists base-level bugs fixed so far and what each fix changes.

## What to customize

Same rule as upstream AK: **do not edit the kernel.** Confine product code to

- `sources/application/app/` - tasks, `task_list.h`, screens, shell commands
- `sources/application/driver/` - board-specific drivers
- `platformio.ini` - `APP_TITLE`, `APP_VERSION` (the only place to set the
  version - see `epcb-platformio-build`), feature defines, include paths

When adding a task, put its row in `app_task_table` (`task_list.cpp`) at the
**same position** as its ID in the `task_list.h` enum: the kernel looks tasks up
by index (`task_table[id]`). Since v1.1.2 a mismatch stops the board at boot
with `FATAL("TK", 0x08)` instead of silently delivering messages to the wrong
task. Wrap `#if` blocks identically in the enum and the table.

Use the `create-task`, `create-driver`, `create-screen` and `use-timer` guides
for the code itself - those are kernel-level and apply unchanged.

## Feeding fixes back

A fix that belongs to every product (kernel port, build flags, BSF tooling) goes
into `ak-base-kit-pio` and gets a new tag. Products then merge it deliberately,
on their own schedule - never automatically.

Product-specific code never goes back into the source base.

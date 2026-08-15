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

Clone a **tagged** version, then drop the source-base history so the product
starts its own:

```bash
git clone --depth 1 --branch v1.0.0 \
  https://github.com/hohoanganh/ak-base-kit-pio.git my-product
cd my-product
rm -rf .git
git init
```

Then, in the product's `README.md`, record the base version:

> Khởi tạo từ ak-base-kit-pio **v1.0.0**

This single line is what makes it possible later to tell which products are
missing a given source-base fix. Without it there is no way back.

Do not work out of a OneDrive-synced folder: OneDrive locks files inside `.git`
and mid-build `.o` files. Keep repos under `C:\Work\` or `D:\dev\`.

## First build

```bash
pio run -e boot -t upload      # 1. bootloader
pio run -e app  -t upload      # 2. application
pio run -e app  -t bsf         # 3. seed BSF - REQUIRED on a blank board
```

Step 3 is not optional; without it the bootloader never jumps to the app and the
board looks bricked. See the `epcb-platformio-build` guide.

## What to customize

Same rule as upstream AK: **do not edit the kernel.** Confine product code to

- `sources/application/app/` - tasks, `task_list.h`, screens, shell commands
- `sources/application/driver/` - board-specific drivers
- `platformio.ini` - `APP_TITLE`, `APP_VERSION`, feature defines, include paths

Use the `create-task`, `create-driver`, `create-screen` and `use-timer` guides
for the code itself - those are kernel-level and apply unchanged.

## Feeding fixes back

A fix that belongs to every product (kernel port, build flags, BSF tooling) goes
into `ak-base-kit-pio` and gets a new tag. Products then merge it deliberately,
on their own schedule - never automatically.

Product-specific code never goes back into the source base.

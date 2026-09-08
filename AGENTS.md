# Repository Guidelines

## Project Structure & Module Organization

This workspace contains three related repositories:

- `bao-hypervisor/` is the Bao core. C11 sources are under `src/core` and
  `src/lib`; architecture code is in `src/arch`; board support and drivers are
  in `src/platform`; configurations and generators are in `configs/` and
  `scripts/`.
- `bao-demos/` contains guest recipes, VM configurations, device trees, and
  platform deployment instructions. Keep platform-specific files under the
  matching `platforms/<name>/` and `demos/<name>/` directories.
- `arm-trusted-firmware/` is the upstream firmware tree with its own rules;
  keep firmware-only changes isolated there.

Generated files belong in `bao-hypervisor/build/`, `bao-hypervisor/bin/`, or
`bao-demos/wrkdir/` and must not be committed.

`bao-hypervisor` and `bao-demos` are intentionally separate. The former owns
the hypervisor core and platform code; the latter assembles guest operating
systems, device trees, initramfs images, and deployment artifacts. The AM625
demo uses the local hypervisor checkout, while Linux and rootfs sources remain
in `/home/wanguo/workspace`.

## Build, Test, and Development Commands

Bao requires a target cross-compiler (for example, `aarch64-none-elf-`). From
`bao-hypervisor/`, build a configuration with:

```sh
make PLATFORM=qemu-aarch64-virt CONFIG=null
```

For the AM625 ALIENTEK bring-up, `/home/wanguo/workspace` is the Buildroot/BSP
reference and should not be modified. From `bao-demos/`, build the Linux demo
with the BSP toolchain; this creates `wrkdir/imgs/am625-alientek/linux/bao.bin`:

```sh
make PLATFORM=am625-alientek DEMO=linux \
  CROSS_COMPILE=/opt/buildroot/aarch64-ca53-linux-gnu_sdk-buildroot/bin/aarch64-ca53-linux-gnu- \
  NO_INSTRUCTIONS=1
```

Use `make ... clean` to remove target outputs and `make ci` for license/format
checks. `make run` applies only to supported QEMU targets; `pandoc` is required
for printed demo instructions, or use `NO_INSTRUCTIONS=1`.

The dual `linux+linux` demo builds separate initramfs images from the same BSP
inputs. Linux0 runs the `bao-iodispatcher` module and `bao-virtio-dm` backend;
Linux1 is log-only and sends its `hvc0` output over VirtIO-console. The backend
PTY is forwarded to Linux0's `/dev/console`, so no second physical UART is
required. The first build needs writable Cargo storage and network access for
the Bao Linux driver and VirtIO device-model repositories.

## AM625 Board Bring-up

The first supported board is ALIENTEK ATK-DLAM62xB (`am625-alientek`), with
four Cortex-A53 CPUs and Bao hand-off at `0x82000000`. At U-Boot, load and
start the image with `fatload mmc 0:1 0x82000000 bao.bin` then `go 0x82000000`.
Bao reserves the low 32 MiB; the dual demo partitions DDR between Linux0
(`0x88000000-0xA7FFFFFF`) and Linux1 (`0xA8000000-0xB7FFFFFF`). Linux0 keeps
MAIN_UART0 as the host-facing console; Linux1 has no physical UART or
interactive shell in the VirtIO demo.
Record serial logs and toolchain/BSP versions for hardware results. Do not add
AM6254ATL assumptions until this board boots reliably.

## Coding Style & Naming Conventions

Use four-space C indentation, function-line braces, lowercase `snake_case`,
SPDX headers, and tabs for Makefile recipes. Run the format checker; do not
reformat assembly sections marked `clang-format off`.

## Testing Guidelines

There is no standalone unit-test suite. Treat successful cross-compilation plus
`format-check`, `license-check`, and `gitlint` as the baseline; boot affected
demos on QEMU or hardware when behavior changes. Keep tests beside their
owning component and document toolchain prerequisites.

## Commit & Pull Request Guidelines

Use focused Conventional Commit subjects such as `fix(armv8): ...` or
`feat(build): ...`; include `Signed-off-by:` when required. Pull requests must
state the affected platform, validation commands/results, required hardware,
and serial logs or screenshots for demo changes.

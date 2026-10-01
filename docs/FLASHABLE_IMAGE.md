# Building a Flashable SD Image

This repo contains only u-boot, the kernel, and packaging scripts -- no root
filesystem. A ready-to-flash image is produced by combining this fork's
kernel/u-boot with Banana Pi's official Debian 10 Buster Lite (64-bit) base
image, the only base still published for this board's kernel line (RTD1395
never moved past 4.9.119; Armbian does not support this SoC at all).

The overlay is built and verified by `.github/workflows/build-sd-image.yml`.

## Base image

`2019-08-02-debian-10-buster-lite-preview-aarch64-bpi-w2-m4-sd-emmc.img`
(850MB zipped, 7.65GB raw), downloaded from a Google Drive link checksummed
in the workflow. DOS partition table:

- 100MiB raw gap before partition 1 -- this is where the bootloader gets
  embedded (see below), confirmed by `scripts/bootloader.sh`'s own byte math.
- Partition 1: 256MiB FAT32, starts at sector 204800.
- Partition 2: ext4, starts at sector 729088.

The base image ships kernels/modules for multiple unrelated boards
(`5.2.5-sunxi64` Armbian/Allwinner artifacts under `/boot` and
`/lib/modules` on the root partition, plus a `BPI-W2` kernel) -- these are
vestigial scaffolding from whatever generic ARM64 Debian rootfs Banana Pi
started from, not anything this board's u-boot actually boots. Leave them
alone; only the `4.9.119-BPI-M4-Kernel` module directory and the
`bananapi/bpi-m4/linux/` directory on the boot partition matter here.

## What gets overlaid, and where

The boot partition's `bananapi/bpi-m4/linux/` directory is the real
per-board boot path (this board's u-boot reads from here, not any generic
`uEnv.txt`/`extlinux.conf` at the FAT32 root -- there isn't one). It
contains:

| File | Overlay action |
|---|---|
| `uImage` | **Replace** with this fork's `linux-rtk/arch/arm64/boot/Image`, renamed. Despite the name, this is just the raw kernel `Image` -- confirmed from `build.sh`'s own `cp_download_files()`, which does exactly `cp Image uImage` with no `mkimage` wrapping. Don't wrap it; arm64's own kernel build has no native `uImage` target anyway. |
| `rtd-1395-bananapi-m4.dtb` | **Replace** with this fork's built dtb. |
| `u-boot-bpi-m4.bin` | **Replace** with this fork's `u-boot-rtk/u-boot.bin`. |
| `uEnv.txt`, `uInitrd`, `bluecore.audio`, `bluecore.audio.enc.A01` | **Leave untouched.** `linux-rtk/` in this fork is byte-identical to upstream (`BPI-SINOVOIP/BPI-M4-bsp`), so the existing initrd stays compatible. The bluecore files are Realtek DSP/audio-core firmware blobs, unrelated to kernel/u-boot. |

On the root (ext4) partition: replace `/lib/modules/4.9.119-BPI-M4-Kernel`
entirely with this fork's freshly built modules (same kernel release string,
confirmed from `linux-rtk/Makefile`'s VERSION/PATCHLEVEL/SUBLEVEL plus the
defconfig's `CONFIG_LOCALVERSION="-BPI-M4-Kernel"` with
`CONFIG_LOCALVERSION_AUTO` unset -- no surprise git-hash suffix).

## The bootloader raw write

The RTD1395 boot ROM reads a small bootloader blob from a fixed raw byte
offset near the start of the device -- before any filesystem, which is why
partition 1 starts 100MiB in. `scripts/bootloader.sh` (invoked by `make
pack`) produces this blob: it writes `u-boot.bin` at offset 40KiB into a
scratch buffer, then extracts that buffer starting from its own 2KiB mark.
So byte 0 of the resulting `*-2k.img.gz` blob corresponds to **absolute
offset 2KiB** on the target device (40KiB written minus the 2KiB trim point
= 38KiB into the blob; 2KiB + 38KiB = 40KiB absolute, matching where it was
originally written). The workflow writes it there with
`dd ... bs=1024 seek=2 conv=notrunc`.

This offset is derived from the vendor script's own math, not from official
documentation -- it's the single most hardware-sensitive, least-independently-
verified number in this whole pipeline. If a built image doesn't boot, check
this first.

## Gaps in the vendor's own build scripts, worked around in CI

- `scripts/bootloader.sh` expects `u-boot.bin` at
  `rtk-pack/rtk/<product>/bin/u-boot.bin`, but nothing in this repo -- not
  the Makefile, not `build.sh`, not `mk_pack.sh` -- ever creates that path
  or copies a file there. Running `make pack` on a clean checkout fails
  outright. The workflow copies `u-boot-rtk/u-boot.bin` there itself before
  calling `make pack`.
- `bootloader.sh` uses `losetup`/`dd` to build the blob, which needs loop
  device access a default Docker container doesn't have. The build job runs
  the vendor's Docker image with `--privileged`.
- `bootloader.sh`'s own output lands at `/tmp/<product>/*-2k.img.gz` --
  inside the container, that's the container's ephemeral `/tmp`, destroyed
  when `--rm` cleans up. The workflow mounts a host directory over `/tmp` so
  the output actually survives the container exiting.

## What's still unverified

Nothing here has been boot-tested on real hardware -- everything above is
derived from the vendor's own scripts and real `inspect_partitions` output,
not from a confirmed-working flash. Treat the first real test boot as the
actual validation step, not this document.

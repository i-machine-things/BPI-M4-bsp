# Coding Notes

Standards and practices reference for this repo — a log of coding patterns and past findings, grouped by topic. Referenced by `.claude/CLAUDE.md` Rule 0 and Rule 4.

## CI

- The vendor build image `sinovoip/bpi-build-linux-4.4` on Docker Hub has no `:latest` tag — its only published tag is `:ubuntu16.04`. Always pin it explicitly.
- `gdown` removed the `--id FILEID` flag; pass the Google Drive file ID as the positional argument instead (`gdown FILEID -O dest`).
- `losetup -fP` returns before udev creates the `p1`/`p2` partition device nodes; `udevadm settle` + poll for both `-b` nodes before mounting, or you race a mount against nodes that do not exist yet.
- **A commit message describing what code does is not proof the code is there — diff the actual PR, don't trust the prose.** A previous session's PR claimed to add mount-and-report partition logic (three commits: the feature, a race-condition fix, a docs note); checking `gh api .../pulls/N/files` directly, only the docs note ever landed. The described feature code was never committed. Always check `gh pr view <n> --json files` or the real diff before building on top of "already done" work from a commit message alone.
- **`docker run` without `--privileged` can't create loop devices.** Any step that needs `losetup`/`mount` inside a container (e.g. `scripts/bootloader.sh`'s own `losetup`+`dd`) needs `--privileged` on the `docker run` invocation, not just inside the guest script.
- **A script's internal `/tmp` usage doesn't survive `docker run --rm` unless `/tmp` itself is a mounted volume.** `scripts/bootloader.sh` writes its real output to `/tmp/<product>/*-2k.img.gz` — that's the container's own ephemeral `/tmp`, destroyed when the container is removed. Mount a host directory over `/tmp` (`-v host/path:/tmp`) if a script's output location is hardcoded under it and you need that output afterward.
- **`scripts/bootloader.sh` expects `u-boot.bin` at `rtk-pack/rtk/<product>/bin/u-boot.bin`, but nothing in this repo ever creates that path.** Not the Makefile, not `build.sh`, not `mk_pack.sh`. Running `make pack` on a clean checkout fails outright. Copy the freshly-built `u-boot-rtk/u-boot.bin` there manually before calling `make pack` — this is a genuine gap in the vendor's own build scripts, not something to "fix" by modifying the vendor script itself.
- **The boot partition's `uImage` file is just the raw kernel `Image`, renamed — not `mkimage`-wrapped.** Confirmed from `build.sh`'s own `cp_download_files()` (`cp .../Image .../uImage`, no `mkimage` call anywhere in the chain). arm64's own kernel build has no native `uImage` make target at all; don't assume one exists or try to wrap the image yourself just because the destination filename says "uImage".

# Coding Notes

Standards and practices reference for this repo — a log of coding patterns and past findings, grouped by topic. Referenced by `.claude/CLAUDE.md` Rule 0 and Rule 4.

## CI

- The vendor build image `sinovoip/bpi-build-linux-4.4` on Docker Hub has no `:latest` tag — its only published tag is `:ubuntu16.04`. Always pin it explicitly.
- `gdown` removed the `--id FILEID` flag; pass the Google Drive file ID as the positional argument instead (`gdown FILEID -O dest`).
- `losetup -fP` returns before udev creates the `p1`/`p2` partition device nodes; `udevadm settle` + poll for both `-b` nodes before mounting, or you race a mount against nodes that do not exist yet.

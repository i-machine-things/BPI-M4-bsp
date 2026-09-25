# Coding Notes

Standards and practices reference for this repo — a log of coding patterns and past findings, grouped by topic. Referenced by `.claude/CLAUDE.md` Rule 0 and Rule 4.

## CI

- The vendor build image `sinovoip/bpi-build-linux-4.4` on Docker Hub has no `:latest` tag — its only published tag is `:ubuntu16.04`. Always pin it explicitly.

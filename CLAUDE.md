# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

This repository is a thin GitHub Actions orchestration layer for building and publishing OpenWrt/iStoreOS firmware for FriendlyElec NanoPi R4S/R5S/R76S, generic ARMv8, and x86_64 targets. It does not contain the OpenWrt source tree or a local package/build system; most workflows create `/builder/openwrt` at CI runtime by running an external build script from the `script_url_general` GitHub secret, while `build-istoreos.yml` clones `istoreos/istoreos` directly.

The tracked files are primarily:

- `README.md`: public user-facing firmware information, download links, default login details, flashing instructions, feature/plugin matrix, and upstream/source links.
- `.github/workflows/*.yml`: CI workflows that compile, package, release, and upload firmware artifacts.

## Common commands

There is no local application build, lint, or test command in this repository. Most validation and all firmware builds happen in GitHub Actions and require repository secrets.

```bash
# Show tracked files in this small orchestration repo
git ls-files

# Basic local sanity checks for documentation/workflow edits
git diff --check -- README.md CLAUDE.md .github/workflows/*.yml
git diff -- README.md CLAUDE.md .github/workflows/*.yml

# List available Actions workflows
gh workflow list

# Trigger a full workflow matrix on a branch/ref
gh workflow run build-release.yml --ref main
gh workflow run build-std.yml --ref main
gh workflow run build-minimal.yml --ref main
gh workflow run build-snapshots.yml --ref main
gh workflow run openwrt-core.yml --ref main
gh workflow run build-istoreos.yml --ref main

# Inspect and follow workflow runs
gh run list --workflow build-release.yml --limit 10
gh run watch <run-id>
gh run view <run-id> --log-failed

# After a matrix workflow has run, rerun only failed jobs or one job
gh run rerun <run-id> --failed
gh run view <run-id> --json jobs --jq '.jobs[] | {name, databaseId, conclusion}'
gh run rerun --job <job-database-id>
```

Single-test note: the workflows do not define `workflow_dispatch` inputs for selecting one matrix model/version. A manual workflow dispatch runs the full matrix; use `gh run rerun --job <job-database-id>` only after a run exists, or add a temporary workflow/matrix change on a branch if an isolated target build is required.

## Workflow architecture

All firmware workflows run on `ubuntu-24.04` with Bash, set the timezone to `Asia/Shanghai`, free `/builder` disk space, install the OpenWrt build environment via `sbwml/actions@openwrt-build-setup`, and install LLVM via `sbwml/actions@install-llvm`. The OpenWrt release/standard/minimal/snapshot/core workflows compile from `/builder` using:

```bash
bash <(curl -sS ${{ secrets.script_url_general }}) <tag-type> <model>
```

`build-istoreos.yml` instead clones `istoreos/istoreos` directly and then reuses this project's public prepare scripts/config fragments. Common compile flags include Clang kernel LTO, ccache, BPF, LTO, LRNG, GCC 16 for OpenWrt 25.12 workflows, GCC 14 for iStoreOS 24.10, and mold. On compile failure the workflows enter `/builder/openwrt` and run `make V=s IGNORE_ERRORS="n m"` for verbose diagnostics.

### Release channels

- `build-release.yml`: full release build. Triggered manually or by `repository_dispatch` type `release`. Uses `type: rc2`, `version: openwrt-25.12`, creates/updates the latest GitHub release using the OpenWrt tag, uploads firmware to FTP under `release/<model>/`, uploads OTA files, and triggers the external aDrive upload helper for non-ARMv8 models.
- `build-std.yml`: standard release build. Triggered manually or by `repository_dispatch` type `std`. Sets `STD_BUILD=y`, creates a non-latest `-standard` release, uploads to `standard/<model>/`, and publishes OTA files. The release body describes this channel as excluding Docker while retaining HomeProxy/common plugins.
- `build-minimal.yml`: minimal release build. Triggered manually or by `repository_dispatch` type `minimal`. Sets `MINIMAL_BUILD=y`, creates a non-latest `-minimal` release, uploads artifacts to GitHub Actions and FTP under `minimal/<model>/`, publishes OTA files, and triggers the minimal aDrive upload helper for non-ARMv8 models.
- `build-snapshots.yml`: nightly/manual snapshot build. Triggered by `workflow_dispatch` and scheduled daily at `0 16 * * *` UTC. Uses `type: dev`, `version: openwrt-25.12`, uploads GitHub Actions artifacts, conditionally uploads to Aliyunpan during the configured hour window, then syncs FTP under `snapshot/<model>/`.
- `openwrt-core.yml`: kernel module/core package build. Triggered manually. Builds a reduced matrix (`armv8`, `nanopi-r5s`, `x86_64`) with `OPENWRT_CORE=y`, publishes `kmod-openwrt-25.12` as a prerelease, dispatches `sbwml/openwrt_core3`, and maintains ccache.
- `build-istoreos.yml`: iStoreOS build. Triggered manually or by `repository_dispatch` type `istoreos`. Clones `https://github.com/istoreos/istoreos.git` at `istoreos-24.10`, supports `nanopi-r4s`, `nanopi-r5s`, and `x86_64`, reuses this project's public prepare/config fragments, and enables OTA, BPF, LTO, LRNG, Mold, GCC 14 (highest version available in upstream iStoreOS 24.10), Clang kernel LTO, ccache, and iStore packages.

### Target/model handling

Most firmware workflows build this model matrix:

- `armv8`: outputs from `openwrt/bin/targets/armsr/armv8*`, including kernels, rootfs, combined EFI images, and `u-boot-qemu_armv8` tarball.
- `nanopi-r4s`, `nanopi-r5s`, `nanopi-r76s`: outputs from `openwrt/bin/targets/rockchip/*`, mainly `*.img.gz`, manifest, and `config.buildinfo`.
- `x86_64`: outputs from `openwrt/bin/targets/x86/*`, including ext4/squashfs combined EFI images, generic rootfs tarball, manifest, and `config.buildinfo`.

The NanoPi targets share the `nanopi-r5s` ccache key (`openwrt-25.12-nanopi-r5s`) to improve cache reuse. Non-NanoPi targets use their model name in the cache key. The iStoreOS workflow uses `istoreos-24.10` cache keys and intentionally omits `armv8` and `nanopi-r76s` because the upstream `istoreos-24.10` branch has matching R4S/R5S/x86_64 targets but no NanoPi R76S target in `target/linux/rockchip/image/armv8.mk`.

### Artifact flow

After compilation, workflows assemble a `rom/` directory and usually an `info/` directory:

1. Copy firmware images from `/builder/openwrt/bin/targets/...`.
2. Copy `manifest.txt` and `config.buildinfo`.
3. Generate `sha256sums.txt` from the selected artifacts.
4. Optionally package build info as `buildinfo_<model>.tar.gz` for GitHub releases.
5. Touch `.ftp-deploy-sync-state.json` before FTP sync where the deploy action expects it.
6. Publish to GitHub Releases, GitHub Actions artifacts, FTP, OTA directories, Aliyunpan, or external upload helpers depending on the channel.

When changing artifact patterns, keep the target-specific output paths and manifest filename patterns in sync across the corresponding release, standard, minimal, and snapshot workflows.

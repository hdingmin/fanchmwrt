# AGENTS.md

FanchmWrt is an OpenWrt fork that adds custom firewall/app-filtering features.
**All custom code lives in `package/fcm/`** — everything else is near-stock upstream OpenWrt and should generally not be touched unless you are porting an upstream change.

## Build system

Standard OpenWrt buildroot (GNU make, Linux host only; Ubuntu 22 recommended).
The build dir path **must not contain spaces** (enforced in `Makefile`).

First-time setup, in order:

```sh
./scripts/feeds update -a
./scripts/feeds install -a      # MUST run before any build
make menuconfig                 # or: make defconfig
make -j$(nproc)
```

`.config`, `bin/`, `build_dir/`, `staging_dir/`, `tmp/`, `feeds/`, `package/feeds/` are all gitignored generated artifacts — never commit them, never assume they exist.

`feeds.conf.default` pins packages/luci/routing/telephony/video to specific commits; the `fanchmwrt` and `openclash` feeds are **unpinned** (track default/dev branch). The `openclash` feed provides `luci-app-openclash` (Clash/Mihomo proxy LuCI app); its Makefile calls `po2lmo` (built from `luci-base` in the luci feed) for i18n, so luci-base must be built first — guaranteed by `luci-theme-fanchmwrt`'s `+luci-base` dependency. OpenClash also requires `dnsmasq-full` (not the lite `dnsmasq`) — select it in menuconfig.

## The custom packages (`package/fcm/`)

| Package | What it is | Build type |
|---|---|---|
| `fwx/` | kernel module (`fwx.ko`, netfilter extension) | `KernelPackage`, compiled against `$(LINUX_DIR)` via kbuild (`obj-m += fwx.o`) |
| `fwxd/` | userspace C daemon, procd service (`/etc/init.d/fwx`, START=96) | `BuildPackage`, hand-written `src/Makefile` |
| `libfwx_common/` | shared lib (`libfwx_common.so`) consumed by fwxd | `BuildPackage` + `Build/InstallDev` (exposes headers) |
| `luci-theme-fanchmwrt/` | LuCI theme (defaults to zh-Hans) | `luci.mk` from the **luci feed** |

Important constraints an agent would otherwise miss:

- **`luci-theme-fanchmwrt` includes `feeds/luci/luci.mk`** — the luci feed must be installed (`./scripts/feeds install luci`) or the theme package fails to evaluate. This is why `feeds install -a` is mandatory.
- **`fwxd` DEPENDS on `+libfwx_common`** and loads the `fwx` kmod at runtime — libfwx_common must be selected and the fwx kmod present on the target.
- `fwxd` also depends on libsqlite3, libmosquitto, libcurl, libubox/ubus/uci, libjson-c; on x86 it pulls `lm-sensors` (`TARGET_x86:lm-sensors`).
- `fwxd`'s `src/Makefile` lists object files explicitly; adding a new `.c` file means editing that Makefile's `OBJS`.

## Building a single package

```sh
make package/fcm/fwxd/compile V=s        # userspace daemon
make package/fcm/libfwx_common/compile V=s
make package/fcm/fwx/compile V=s          # kernel module (needs kernel built/prepared)
make package/fcm/fwxd/clean V=s
```

`V=s` gives full verbose output — use it when debugging build failures.

Clean levels: `make clean` (build artifacts), `make dirclean` (also toolchain/staging — full reset), `make targetclean`.

## Build-time `fwx_init` step

The top-level `Makefile` has a custom `fwx_init` target (run as part of `prereq`) that, for x86/i386/x86_64/rockchip targets, writes `EXPAND_ROOT=0|1` into `package/base-files/files/etc/product_feature` based on `CONFIG_TARGET_ROOTFS_PARTSIZE` (>300 → 1). If you change rootfs sizing or target handling, this generated file is expected to exist at runtime.

Runtime config flow: `fwx.init` symlinks `/etc/fwxd/feature.cfg` → `/tmp/feature.cfg`; `fwxd` reads both `/tmp/feature.cfg` and `/etc/product_feature`.

## Tests / verification

There is **no unit-test harness for the `fcm` C code**. `tests/Makefile` is optionally included but does not exist. `make check` only runs stamp-check over tools/toolchain/packages — it does not exercise fwx/fwxd. Verification is done by building and running on a target device. `scripts/checkpatch.pl` is available for kernel-style patch/format checks.

## Conventions

- Sign off commits (DCO). `.vscode/settings.json` sets `git.alwaysSignOff: true`; follow OpenWrt's https://openwrt.org/submitting-patches for patches/PRs.
- SPDX-License-Identifier headers are used throughout; project is GPL-2.0.
- The `fwx` kernel module build suppresses many warnings via `EXTRA_CFLAGS` (`-Wno-...`) — do not "fix" these by removing the flags; the kernel build is strict and upstream-incompatible code relies on them.

## CI

Workflows reuse `openwrt/actions-shared-workflows`:
- `formal.yml` — PR formalities checks.
- `packages.yml` / `kernel.yml` / `tools.yml` / `toolchain.yml` — build on PR/push touching the relevant paths (`package/**`, `target/linux/**`, `tools/**`, `include/**`, `toolchain/**`).
- `build-on-comment.yml` — OpenWrt reviewers can comment `build <target>/<subtarget>/<profile>` on a PR to trigger a full firmware build (writes `CONFIG_TARGET_*` lines, then `make defconfig && make -j$(nproc) BUILD_LOG=1`; uploads `logs/` and `bin/` artifacts).
- CI/dev container image: `ghcr.io/openwrt/buildbot/buildworker-v3.8.0:v9` (see `.devcontainer/ci-env/devcontainer.json`).

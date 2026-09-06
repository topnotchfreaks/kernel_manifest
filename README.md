# Xiaomi Topaz (SM6225/bengal) — GKI Kernel Build Guide

Build guide for the `msm-kernel` android13-5.15 GKI tree targeting the Xiaomi Topaz (Qualcomm SM6225 / bengal platform).

## Prerequisites

- Linux build host with `clang`/LLVM toolchain support
- `repo`/`git` for source sync
- Sufficient disk space for kernel + vendor module sources (~15GB+ recommended)

## Repository Layout

Your kernel workspace root (referred to as `KITCHEN` below) must contain these as **sibling directories**:

```
KITCHEN/
├── build/                          # Kernel build scripts (build.sh, gettop.sh, etc.)
├── common/                         # ACK GKI common kernel — android13-5.15 branch
├── msm-kernel/                     # Qualcomm SoC kernel tree (topaz/bengal)
└── vendor/
    └── qcom/
        └── opensource/
            ├── audio-kernel/
            ├── camera-kernel/
            ├── dataipa/
            ├── datarmnet/
            ├── datarmnet-ext/       # check for per-feature subdirs (perf, shs, sch, etc.)
            ├── display-drivers/
            ├── graphics-kernel/
            ├── mmrm-driver/
            ├── securemsm-kernel/
            ├── touch-drivers/
            ├── video-driver/
            └── wlan/                # check for qcacld-3.0/ subdir
```

### External vendor modules

Verify each `vendor/qcom/opensource/*` repo actually has a buildable module at the path you reference — some (`wlan`, `datarmnet-ext`) nest the real Kbuild one level deeper:

```bash
for m in audio-kernel camera-kernel dataipa datarmnet datarmnet-ext display-drivers graphics-kernel mmrm-driver securemsm-kernel touch-drivers video-driver wlan; do
  find "vendor/qcom/opensource/$m" -maxdepth 2 -iname "Kbuild" -o -iname "Makefile" 2>/dev/null
done
```
If a search comes back empty, point `EXT_MODULES` at the actual subdirectory (e.g. `vendor/qcom/opensource/wlan/qcacld-3.0`) instead of the parent.

## Building

Run from the `KITCHEN` root:

```bash
LTO=thin VARIANT=gki BUILD_CONFIG=msm-kernel/build.config.msm.topaz ./build/build.sh
```

- `VARIANT=gki` is required — omitting it defaults to `consolidate`, which will **not** produce a GKI-compliant boot image.
- The build internally recurses into `common/` first (base ACK GKI kernel), then builds `msm-kernel` + all `EXT_MODULES` against it, then runs `DIST_CMDS` to package everything.

### Verbose/debug output

If the build stops with no clear error:

```bash
bash -x build/build.sh 2>&1 | tee /tmp/build.log
```
(Remember: env vars must precede `bash -x`, not be dropped when you add it — `LTO=thin VARIANT=gki BUILD_CONFIG=... bash -x build/build.sh ...`)

## Output

After a successful build, check:

```bash
ls -la out/msm-kernel-topaz-gki/dist/
```

Expected artifacts:
- `Image` — the kernel binary
- `dtbo.img` / `*.dtb` — devicetree blobs (via `make_dtbo_img` / `install_dtbs`)
- `vendor_dlkm` modules — from `prepare_vendor_dlkm`, driven by `msm-kernel/modules.list.msm.topaz`
- `system_dlkm` image — from `prepare_system_dlkm`, GKI-common modules

Devicetree and vendor modules are **not** separate repos for this device — they're built in-tree from `msm-kernel/` (DTs under `vendor/qcom`, DLKM module list in `modules.list.msm.topaz`) plus the external module repos above, all packaged in the same `build.sh` run via `MAKE_GOALS`/`DIST_CMDS`.

## Troubleshooting

| Symptom | Likely Cause |
|---|---|
| `_setup_env.sh: ... build.config: No such file or directory` | `BUILD_CONFIG=` was dropped from the command (common when adding `bash -x` after the fact) |
| Nested `env -i bash -c '... build.sh'` fails with no output | `common/` directory missing at `KITCHEN` root |
| `ERROR! Detected overridden config!` | Fragment tries to demote a GKI-required `y` symbol to `m` — remove the override from `vendor/topaz_GKI.config` |
| External module fails to find Kbuild | Repo path needs a deeper subdir (e.g. `wlan/qcacld-3.0`) — verify with `find <repo> -maxdepth 2 -iname Kbuild` |

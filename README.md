# star2lte-los22 — LineageOS 22.2 (Android 15) for Galaxy S9+ (Exynos 9810)

**Skyshadow2020** is the working repo for this project. The first cut of this
pipeline lived at `Skyshadow2022/star2lte-los22` — that repo is a **read-only
reference** now; all development happens here.

Mission (beyond a plain LOS 22.2 build):

1. **Kernel with SUSFS v2 + KernelSU** — our own kernel fork, proven-stable
   parts ported from the `kernel-s9plus` line (audio boot-race fix included).
   KernelSU Manager must install and work out of the box.
2. **Unencrypted shared /data** — the ROM ships with FBE disabled so the same
   data is visible to the ROM and to a TWRP-based recovery, no decrypt stack.
3. **TWRP-based companion recovery** that reads that unencrypted /data.
4. **Samsung app support** — Gallery, Camera, Notes and Samsung Account only.

## What's here

| File | Purpose |
|---|---|
| `.repo-local-manifests/star2lte.xml` | the local manifest (device/kernel/vendor/HALs) |
| `.github/workflows/build.yml` | `repo init` → sync → `mka bacon` → upload the ROM zip |
| `HANDOFF.md` | project state, decisions, bug log, roadmap — read this first |

## Sources it pulls (all verified to exist @ the pinned branch)

- `ExyHyperBrick/android_device_samsung_star2lte` @ `lineage-22.2`
- `ExyHyperBrick/android_device_samsung_exynos9810-common` @ `lineage-22.2`
- `ExyHyperBrick/android_kernel_samsung_exynos9810` @ `lineage-22.2` (4.9.337)
- `ExyHyperBrick/proprietary_vendor_samsung_{star2lte,exynos9810-common}` @ `lineage-22.2`
- `LineageOS/android_hardware_samsung` @ `lineage-22.2` (provides `dtbhtoolExynos`)
- `LineageOS/android_hardware_samsung_slsi-linaro_{exynos,exynos5,interfaces,openmax}` + `…_slsi_sepolicy` @ `lineage-22.2`
- `ExyHyperBrick/android_hardware_samsung_slsi-linaro_{config,graphics}` @ `lineage-22.2` (his forks, per his own `lineage-22.2-wip` manifest)

The manifest mirrors ExyHyperBrick's own `lineage-22.2-wip` roomservice.xml —
the authoritative reference for what builds at 22.2. IMS/VoLTE (krazey
ImsStack/ImsMedia) is **left out of the first build** — add it back after a
plain build boots.

## How to build

**On GitHub Actions:** Actions → *LineageOS 22.2 star2lte* → *Run workflow*.
- `mode: build` — full sync + compile.
- `mode: sync` — fetch source only (useful to validate manifest changes fast).

The finished `lineage-*.zip` (+ `boot.img`/`recovery.img`) is uploaded as the
run artifact.

### ⚠️ Free-runner limits (read this)

A cold, from-scratch LineageOS 22 build is heavy:
- **Disk:** the `maximize-build-space` step reclaims ~65-75 GB; a shallow sync
  keeps the tree ~30-40 GB and the build needs another ~30-50 GB. It fits, but
  with little headroom.
- **Time:** free GitHub-hosted runners cap a job at **6 hours** on 2-4 vCPUs.
  A cold full build can exceed that. `ccache` is persisted between runs, so a
  second run is much faster; if the first build times out, re-run it — object
  files that finished are cached.

## Local manifest, standalone

Outside CI, to build anywhere:
```bash
repo init -u https://github.com/LineageOS/android.git -b lineage-22.2 --git-lfs
mkdir -p .repo/local_manifests
cp .repo-local-manifests/star2lte.xml .repo/local_manifests/
repo sync -c --no-clone-bundle -j$(nproc)
source build/envsetup.sh && brunch star2lte
```

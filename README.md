# star2lte-los22 — LineageOS 22.2 (Android 15) for Galaxy S9+ (Exynos 9810)

A build pipeline for **LineageOS 22.2** on `star2lte` (SM-G965F), driven from
GitHub Actions. It pins the community trees maintained by **ExyHyperBrick**
(the developer behind the XDA LOS 23.2/24.0 builds) at their `lineage-22.2`
branches, plus the LineageOS Exynos HALs at `lineage-22.2`.

## What's here

| File | Purpose |
|---|---|
| `.repo-local-manifests/star2lte.xml` | the local manifest (device/kernel/vendor/HALs) |
| `.github/workflows/build.yml` | `repo init` → sync → `mka bacon` → upload the ROM zip |

## Sources it pulls

- `ExyHyperBrick/android_device_samsung_star2lte` @ `lineage-22.2`
- `ExyHyperBrick/android_device_samsung_exynos9810-common` @ `lineage-22.2`
- `ExyHyperBrick/android_kernel_samsung_exynos9810` @ `lineage-22.2`
- `ExyHyperBrick/proprietary_vendor_samsung_{star2lte,exynos9810-common}` @ `lineage-22.2`
- `LineageOS/android_hardware_samsung*` + `…_slsi_sepolicy` @ `lineage-22.2`
- `ExyHyperBrick/…_exynos_dtbh` @ `lineage-23.2` (a version-independent build tool)

IMS/VoLTE (krazey ImsStack/ImsMedia) is **left out of this first build** — add
it back after a plain build boots.

## How to build

**On GitHub Actions:** Actions → *LineageOS 22.2 star2lte* → *Run workflow*.
- `mode: build` — full sync + compile.
- `mode: sync` — fetch source only (use this first; a full build may not fit
  the free runner, see below).

The finished `lineage-*.zip` (+ `boot.img`/`recovery.img`) is uploaded as the
run artifact.

### ⚠️ Free-runner limits (read this)

A cold, from-scratch LineageOS 22 build is heavy:
- **Disk:** the `maximize-build-space` step reclaims ~65-75 GB; a shallow sync
  keeps the tree ~30-40 GB and the build needs another ~30-50 GB. It fits, but
  with little headroom.
- **Time:** free GitHub-hosted runners cap a job at **6 hours** on 2-4 vCPUs. A
  cold full build can exceed that. `ccache` is persisted between runs, so a
  second run is much faster; if the first build times out, re-run it — object
  files that finished are cached.
- If it still doesn't fit, point the workflow's `runs-on` at a **self-hosted
  runner** (e.g. the VPS: ≥8 cores, ≥16 GB RAM, ≥250 GB disk) — the steps are
  identical there and none of the free-runner limits apply.

## After the build — restore your data

Your pre-migration backup is at `D:\star2lte-backup\2026-09-23\` (photos, SMS,
contacts, call log, Telegram/Instagram). Its `README-restore.md` has the
per-item restore steps for LOS 22.

## Local manifest, standalone

Outside CI, to build anywhere:
```bash
repo init -u https://github.com/LineageOS/android.git -b lineage-22.2 --git-lfs
mkdir -p .repo/local_manifests
cp .repo-local-manifests/star2lte.xml .repo/local_manifests/
repo sync -c --no-clone-bundle -j$(nproc)
source build/envsetup.sh && brunch star2lte
```

# HANDOFF — star2lte LOS 22.2 (Skyshadow2020/star2lte-los22)

Read this before touching anything. Last updated: 2026-09-24.

## Mission

Build **LineageOS 22.2 (Android 15)** for Galaxy S9+ Exynos (`star2lte`,
SM-G965F, Exynos 9810 — the Snapdragon variant `star2qltechn` is a DIFFERENT
device) with:

1. **Kernel**: our own fork with **SUSFS v2 + KernelSU**, daily-driver stable,
   KernelSU Manager installs and works with no tricks. Port proven pieces from
   the `kernel-s9plus` line (audio boot-race fix = madera reset polarity +
   supply recycle, 12/12 winning boots).
2. **Data**: **unencrypted /data** — ROM and recovery share one readable data
   partition. FBE must be disabled ROM-side.
3. **Recovery**: **TWRP-based**, companion to the ROM, reading the unencrypted
   /data (no decrypt stack needed — simpler than the Skyshadow2022 TWRP 12.1
   FBE project).
4. **Samsung apps**: Gallery, Camera, Notes, Samsung Account — ONLY these.

## Hard rules (set by Mehran, 2026-09-24)

- **Builds via GitHub Actions ONLY, on the Skyshadow2020 account.** No VPS,
  no self-hosted runners.
- **`Skyshadow2022/*` repos are READ-ONLY** — never push, never edit. Read
  freely (source of the first pipeline cut + the proven kernel work).
- Only Mehran + ZCode work in this repo — no parallel-session paranoia, but
  still `git fetch` before pushing.
- Persian is the chat language; docs/code are English.

## Credentials & locations

- Skyshadow2020 PAT: `~/.gh_token_2020` (chmod 600) — full scope incl.
  `workflow`. Used for repo creation, push (https), Actions dispatch,
  artifact download. **Mehran should revoke it if this project ever dies.**
- Skyshadow2022 PAT: `~/.gh_token` (read-only use per the rule above).
- Local prep clone of this repo: `~/Projects/star2lte-los22-sky2020-prep`.
- Proven kernel tree (read source for the port): `~/kernel-s9plus`
  (branch `susfs-v2-experiment`, mirrors `Skyshadow2022/kernel-s9plus-hdmi`).

## Why this repo exists / provenance

`Skyshadow2022/star2lte-los22` (run #1, 2026-09-23) failed at the
`repo init + local manifest` step and never got to sync. This repo is that
pipeline, fixed and owned by Skyshadow2020.

## Bugs found in the 2022 cut (all fixed here)

| # | Bug | Evidence | Fix |
|---|-----|----------|-----|
| 1 | `build.yml` copied the local manifest from `$GITHUB_WORKSPACE/../repo-config/` — checkout with `path: repo-config` lands at `$GITHUB_WORKSPACE/repo-config`, so `cp` hit a nonexistent path | run #1 job log: `cp: cannot stat '/home/runner/work/star2lte-los22/star2lte-los22/../repo-config/...'` | copy from `$GITHUB_WORKSPACE/repo-config/...` |
| 2 | Manifest pinned `LineageOS/android_hardware_samsung_slsi_nfc` @ `lineage-22.2` — repo has NO such branch (only 23.0/23.1/23.2/24.0) → repo sync would die | GitHub API 404 on the branch; ExyHyperBrick's own `lineage-22.2-wip` manifest omits the project entirely | dropped the project |
| 3 | Manifest pinned `ExyHyperBrick/..._exynos_dtbh` @ `lineage-23.2` unnecessarily — at 22.2 the `dtbhtoolExynos` tool (set via `TARGET_CUSTOM_DTBTOOL` in BoardConfigCommon.mk:81) is built from `LineageOS/android_hardware_samsung` @ `lineage-22.2` (`dtbhtool/Android.bp`) | API + file inspection | dropped the project; hardware/samsung provides the tool |
| 4 | config/graphics HALs pulled from LineageOS instead of ExyHyperBrick's 22.2-era forks | ExyHyperBrick's `lineage-22.2-wip` manifest uses HIS forks for `slsi-linaro_config` + `slsi-linaro_graphics` | switched to his forks |

`starlte`/`crownlte` device/vendor trees are NOT in the manifest (checked:
only referenced in comments inside `proprietary-files.txt`).

## Roadmap (each phase = separate CI runs, never mix too many variables)

- **Phase 1 — green vanilla build** (THIS is where we are): fixed pipeline,
  ExyHyperBrick trees as-is, LOS 22.2 zip out of CI. Acceptance: artifact
  downloads, zip flashes via TWRP, device boots.
- **Phase 2 — kernel fork**: fork `ExyHyperBrick/android_kernel_samsung_exynos9810`
  @ `lineage-22.2` to `Skyshadow2020/android_kernel_samsung_exynos9810`; port
  SUSFS v2 + KSU + madera audio fix from `~/kernel-s9plus`; manifest points at
  the fork. KernelSU Manager installs clean.
- **Phase 3 — unencrypted /data**: fstab + props to disable FBE ROM-side;
  validate with a TWRP build reading /data.
- **Phase 4 — TWRP companion recovery** for LOS 22.2 (unencrypted /data).
- **Phase 5 — Samsung apps**: Gallery/Camera/Notes/Account only (blobs from
  the vendor trees + priv-app overlay + needed props/sepolicy).
- **Phase 6 — daily-driver polish**: IMS/VoLTE (krazey ImsStack) if wanted,
  perf tuning, PITFALLS below re-audited.

## Pitfalls carried over from the star2lte kernel/TWRP projects (STILL TRUE)

- NEVER flash an AOSP-format `recovery.img` on Samsung — needs DTBH packing
  (Samsung-format only). The CI's `recovery.img` artifact is NOT flashable.
- Recovery partition flashes go through heimdall/Odin ONLY (SBL bookkeeping);
  `dd` is fine for BOOT but forbidden for RECOVERY.
- The kernel build: NEVER `make clean` on a warm tree; `make dtb.img` +
  `make dtbo.img` flow for DTBH.
- Phone grep is toybox: no `\|` BRE alternation (use `grep -E` with plain
  `|`), no bash process substitution.

## Run log

### Run #1 (2026-09-24) — FAILED: ENOSPC mid-sync
Died in `repo sync` (`android_hardware_samsung_nfc`). Runner = single 145G
disk (NO separate /mnt). No partial clone, no diet.

### Run #2 — cancelled (dispatched accidentally before the fix landed)

### Run #3 — FAILED: ENOSPC again (same repo, ~25 min in)
Partial clone + 7 emulator removes were NOT enough. Log showed the root fs at
**100% / 100M free** right after the maximize step — the easimon action's
tmp PV gets fallocated out of the SAME root fs (no /mnt), overcommitting it.

### Run #4 — FAILED: apt itself (exit 100)
Root reserve 1024M was too small: `E: You don't have enough free space in
/var/cache/apt/archives/` — apt runs AFTER the maximize step ate the root fs.

### Run #7 — CANCELLED: partial clone is latency-bound (3h+, no end in sight)
The no-LVM recipe worked (apt green, reclaim green), but `repo sync` with
`--partial-clone --clone-filter=blob:limit=500K` ran 4.5h+ without finishing.
With the 6h runner ceiling that path is a dead end → cancelled, run #8
(queued full build) cancelled too.

### Run #9 — sync-mode GREEN: sync = 20 min, tree = ~100G
Full-shallow sync (no partial clone) over the dieted manifest: **~20 minutes
wall** and clean. Tree 136G used / 9.3G free — the SOURCE fits, nothing left
for out/.

### Run #10 — build, three lessons
1. **Deleting .repo/project-objects wholesale = trap**: breakfast/roomservice
   re-downloaded the ENTIRE tree (1.5h, +17G per the df monitor) because
   repo's ref state vanished with the gitdirs, then died on a transient
   GnuTLS network error (frameworks/opt/net/wifi tag fetch).
2. **Namespace diet errors**: hardware/samsung's Android.bp imports namespace
   `hardware/google/pixel`; hardware/qcom/sm8150/display imports
   `hardware/google/interfaces`. Both restored (diet 105→103 repos).
3. actions/cache post-save skips on failed jobs by default →
   `save-always: true` added (the multi-run strategy depends on it).

### Run #11 — build with the final disk recipe (current)
Post-sync prune = **pack/idx files only** (refs survive → `repo sync` stays a
fast no-op; ~45G freed for out/). Expected: sync ~20 min → prune → soong +
build with the fork kernel until the 6h wall; ccache saves either way; the
next run continues warm.

### Diet list (105 remove-projects, every name verified vs LOS 22.2 default.xml)
7 emulator-only (goldfish/cuttlefish/emulator/qemu) + 98 more: GKI kernel
prebuilts x12, hardware/google (pixel/gs101/gs201/zuma graphics) x14, the
whole Car/automotive stack x27, TV apps, macOS/Windows toolchains, bazel
rule repos, ABI dumps, packages/modules/Virtualization (3.5G), heavy
test/fuzz tooling (autotest/AFL++/bcc). If a build ever names a missing
project, move that one line back to a <project>.

## Kernel fork (Phase 2) — GREEN BUILD, on Skyshadow2020

`Skyshadow2020/android_kernel_samsung_exynos9810` branch `lineage-22.2`
(API-fork of ExyHyperBrick; 3 commits: c45d4e50fcc port + 81228701414 hooks +
37a933343ed path_umount backport). **Local build: GREEN** (Image.gz 11.3MB,
0 errors, System.map has 215 ksu_ + 67 susfs_ symbols + madera_dev_init).
The LOS manifest now points the kernel project at this fork (commit 4d62139).

Port gotchas learned (all hit, all fixed):
- **KSU manual hooks live in the PRE-HISTORY of kernel-s9plus** (root commit
  carried them) — root..HEAD diffs never show them. The Kbuild greps for
  `ksu_handle_sys_reboot` in kernel/reboot.c and refuses to build otherwise
  ("No hooks were defined"). Port = curated his→ours hunks (ksu-only) into
  fs/{exec,open,read_write}.c, kernel/reboot.c, drivers/input/input.c.
  Skipped on purpose: LOD_SEC, FSCRYPT_SDP, reboot_cpu kstrtoint refactor,
  WRITE_LIFE line, input EXPORT_SYMBOLs.
- **path_umount/can_umount**: KSU's kernel_umount feature needs them on 4.9;
  the Kbuild normally sed-injects them at drivers/kernelsu parse time —
  AFTER fs/namespace.o compiled → fresh trees fail at vmlinux link with
  "undefined reference to path_umount". Committed the backport directly
  (fs/namespace.c + fs/internal.h prototype); Kbuild's grep guards skip it.
- KSU Kbuild pinned (injected block) to KSU_VERSION=3050 → **33250** /
  tag v3.3.0 — Manager v3.3.0 versionCode 33214 is the floor.
- drivers/kernelsu must be a REAL dir (source tree used a symlink into a
  KernelSU-Next mirror — copying the symlink produces a dangling path).

Build recipe (validated): toolchains `~/toolchains/{clang,gcc-arm64}`,
```
make O=$HOME/Projects/fork-out ARCH=arm64 CC=clang \
  CROSS_COMPILE=aarch64-linux-android- CLANG_TRIPLE=aarch64-linux-gnu- \
  KSU_GIT_VERSION=3050 KSU_GIT_VERSION_VALID=1 KSU_GIT_TAG=v3.3.0 \
  exynos9810-star2lte_defconfig && make ... -j4 Image.gz dtbs
```

## Phase-2 kernel port — facts established (2026-09-24)

- `~/kernel-s9plus` (git root; mirrors `Skyshadow2022/kernel-s9plus-hdmi`,
  branch `susfs-v2-experiment`) is a META repo: the kernel tree lives in
  `kernel_source/`, with `configs/{kernelsu,susfs}.fragment`, `module-susfs/`
  (userspace susfs tool), `out_ksu/` (Manager APKs incl. Next v3.3.0),
  `build_gkilike.sh` as the build driver.
- Kernel base = 4.9.337 — SAME sublevel as ExyHyperBrick's lineage-22.2.
- `drivers/kernelsu` (KSU-Next with core/feature/compat) + `fs/susfs.c` +
  `include/linux/susfs{,_def}.h` present in our tree; 13 ksu/susfs commits +
  the audio-race series to port.
- LOS device tree calls for `exynos9810-star2lte_defconfig`
  (ehb-star2lte/BoardConfig.mk:10) — our tree already HAS that defconfig.
- **NO shared git history** between our kernel repo and ExyHyperBrick's
  (merge-base fails; his tree carries ~660k samsung upstream commits, ours is
  a 56-commit line). Port must be file-level: copy the new-file sets
  (drivers/kernelsu, fs/susfs.c, susfs headers, module glue) and 3-way-apply
  our diffs for shared files (fs/*, kernel/*, drivers/{Makefile,Kconfig},
  security/*, defconfig fragments, sound/soc/codecs/madera*).
- Start the fork defconfig from HIS `exynos9810-star2lte_defconfig` (A15-
  tested) and merge in our susfs/ksu fragments, NOT the other way around.

## Next steps (as of this handoff)

1. Run #2 (fixed) dispatched — watch sync `df -h` output for the real source
   footprint with partial clone, then whether the full build fits.

1. Dispatch `mode: build` on this repo, watch the run, fix whatever breaks.
2. In parallel: explore `~/kernel-s9plus` integration layout (where KSU-Next
   and SUSFS live) and draft the port series for Phase 2.

# KernelSU-Next Integration Dossier for Motorola Edge S (`nio`)

Date: 2026-05-07 (UTC)  
Kernel tree: `kernel/motorola/sm8250`  
Device target: Motorola Edge S / SM8250 / `nio` / XT2125-4  
Status: Integrated (post-failure patchset updated, runtime validation pending device test)

## 1. Objective

Integrate KernelSU-Next into the `nio` kernel in a reproducible way, with explicit design decisions, milestone commits, and patch exportability.

## 2. Local Build/Input Topology

From local manifests and device config:

- `.repo/local_manifests/roomservice.xml` contains:
  - `kernel/motorola/sm8250` -> `LineageOS/android_kernel_motorola_sm8250`
  - `device/motorola/nio`
  - `device/motorola/sm8250-common`
- Device kernel config chain:
  - `device/motorola/sm8250-common/BoardConfigCommon.mk`:
    `vendor/kona-perf_defconfig` + `vendor/ext_config/moto-kona.config`
  - `device/motorola/nio/BoardConfig.mk` appends:
    `vendor/ext_config/nio-default.config`

Kernel baseline in this tree:

- `Makefile`: Linux `4.19.325` (non-GKI legacy kernel path for this integration).

## 3. Research Inputs

References used for design and compatibility decisions:

- KernelSU-Next integration doc (non-GKI):  
  `https://kernelsu-next.github.io/webpage/pages/how-to-integrate-for-non-gki.html`
- KernelSU-Next `v3.2.0-legacy`:
  - `kernel/setup.sh`
  - `kernel/Kbuild`
  - `kernel/Kconfig`
- KernelSU-Next repository:
  `https://github.com/KernelSU-Next/KernelSU-Next`
- LineageOS build references:
  - `https://github.com/LineageOS-infra/build-config`
  - `https://wiki.lineageos.org/devices/nio/build/variant1/`

## 4. Design Conclusions

### 4.1 Hook mode selection

Selected **manual hook mode** for this 4.19 kernel.

Rationale:

- `KernelSU-Next/kernel/Kconfig` explicitly states the kprobes hook mode should not be used on kernels below 5.10.
- This tree is 4.19; therefore manual syscall hooks are the conservative path.

### 4.2 Metadata/version strategy

Selected **git submodule** for `KernelSU-Next` at kernel root:

- Path: `KernelSU-Next`
- Pinned ref: `v3.2.0-legacy` (`9b08e88862000d5c50fb2e43a5b75123cf472e54`)

Rationale:

- `KernelSU-Next/kernel/Kbuild` computes version/tag from git metadata and differentiates KernelSU repo root vs kernel repo root.
- Submodule preserves complete KernelSU git history/metadata and avoids fallback version behavior.

### 4.3 Build graph injection

Following upstream `setup.sh` semantics:

- `drivers/kernelsu` symlink -> `../KernelSU-Next/kernel`
- `drivers/Makefile`: `obj-$(CONFIG_KSU) += kernelsu/`
- `drivers/Kconfig`: `source "drivers/kernelsu/Kconfig"`

### 4.4 Kernel hooks added

Manual hook calls were integrated at syscall entry points:

- `fs/exec.c`
  - `ksu_handle_execveat(...)` in `do_execve` and `compat_do_execve`
- `fs/open.c`
  - `ksu_handle_faccessat(...)` in `SYSCALL_DEFINE3(faccessat, ...)`
- `fs/read_write.c`
  - `ksu_handle_sys_read(fd)` in `SYSCALL_DEFINE3(read, ...)` (gated by `ksu_vfs_read_hook`)
- `fs/stat.c`
  - `ksu_handle_stat(...)` in `SYSCALL_DEFINE4(newfstatat, ...)`
- `kernel/reboot.c`
  - `ksu_handle_sys_reboot(...)` in `SYSCALL_DEFINE4(reboot, ...)`

### 4.5 Device config enablement

Added to `arch/arm64/configs/vendor/ext_config/nio-default.config`:

- `CONFIG_KSU=y`
- `CONFIG_KPROBES=y`
- `CONFIG_KPROBE_EVENTS=y`
- `CONFIG_KSU_MANUAL_HOOK=y`
- `# CONFIG_KSU_KPROBES_HOOK is not set`

## 5. Milestones and Commits

1. Milestone M1 (source + build graph)
   - Commit: `764a962f1c2d`
   - Message: `kernel: integrate KernelSU-Next submodule and build wiring`
   - Includes: submodule, `.gitmodules`, `drivers/*` wiring, symlink.

2. Milestone M2 (manual hooks + nio config)
   - Commit: `3f615115c961`
   - Message: `kernel: add KernelSU-Next manual hooks for nio`
   - Includes: syscall hook integration + `nio-default.config` enablement.

3. Milestone M3 (deterministic backports for KSU build ordering)
   - Commit: `b2ef284ff83b`
   - Message: `fs: backport path_umount and seccomp filter_count for KernelSU`
   - Includes:
     - `fs/namespace.c`: adds `can_umount()` + `path_umount()`
     - `fs/internal.h`: adds `extern int path_umount(...)`
     - `include/linux/seccomp.h`: adds `atomic_t filter_count` support
   - Reason:
     - Build failure observed in `out/error.log`:
       `ld.lld: error: undefined symbol: path_umount`
     - Root cause was KernelSU `Kbuild` mutating source files during build after
       compilation order had already produced objects without `path_umount`.
       This milestone makes those backports permanent and deterministic.

## 6. Verification Performed Here

Build/flash/runtime testing was intentionally deferred to device-side validation.

Local static checks completed:

- Verified merged config accepts KSU options via:
  - `scripts/kconfig/merge_config.sh -m -O /tmp/ksu_nio_cfg ...`
- Confirmed merged output includes:
  - `CONFIG_KSU=y`
  - `CONFIG_KSU_MANUAL_HOOK=y`
  - `CONFIG_KSU_KPROBES_HOOK` unset

No full kernel build or boot test was executed in this operation.
One user-provided build failure log was analyzed and fixed in-source (M3).

## 7. Operational Notes for Future Sync

After syncing or clean checkout, ensure submodule materialization:

1. `cd kernel/motorola/sm8250`
2. `git submodule sync --recursive`
3. `git submodule update --init --recursive`
4. `cd KernelSU-Next && git checkout v3.2.0-legacy`

## 8. Rollback

To roll back integration cleanly:

1. `git revert 3f615115c961`
2. `git revert 764a962f1c2d`
3. `git revert b2ef284ff83b`

Or reset branch to pre-integration commit if appropriate for your workflow.

## 9. Expected Device-Side Test Focus

When you validate on hardware:

- Boot success and SELinux state transition stability.
- Manager install/handshake via KernelSU-Next manager.
- `su` compatibility behavior.
- No reboot syscall regressions.
- No regressions in `faccessat`, `newfstatat`, and regular `read` path.

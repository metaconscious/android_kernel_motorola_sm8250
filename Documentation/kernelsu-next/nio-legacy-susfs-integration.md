# KernelSU-Next `legacy-susfs` + SuSFS Integration Dossier for Motorola Edge S (`nio`)

Date: 2026-05-07 (UTC)  
Kernel tree: `kernel/motorola/sm8250`  
Device: Motorola Edge S / SM8250 / `nio` / XT2125-4  
Baseline branch before this work: `ksu-next-nio-integration`

## 1. Objective

Switch KernelSU-Next from `v3.2.0-legacy` to `legacy-susfs`, integrate SuSFS kernel-side changes for Linux 4.19 non-GKI, and preserve reproducible metadata/version behavior required by KernelSU-Next `kernel/Kbuild`.

## 2. References Used

1. KernelSU-Next non-GKI guide:  
   `https://kernelsu-next.github.io/webpage/pages/how-to-integrate-for-non-gki.html`
2. KernelSU-Next repo and branch `legacy-susfs`:  
   `https://github.com/KernelSU-Next/KernelSU-Next`
3. KernelSU-Next `v3.2.0-legacy` `setup.sh` and `Kbuild` behavior.
4. SuSFS official repository (`kernel-4.19` branch):  
   `https://gitlab.com/simonpunk/susfs4ksu`
5. Lineage build references:
   - `https://github.com/LineageOS-infra/build-config`
   - `https://wiki.lineageos.org/devices/nio/build/variant1/`

## 3. Design Decisions

### 3.1 KernelSU source and metadata strategy

- Kept `KernelSU-Next` as a git submodule to preserve tag/history metadata consumed by KernelSU-Next `Kbuild`.
- Moved submodule pointer from `v3.2.0-legacy` to `origin/legacy-susfs` commit `4cc162e027cc36f8e4d4e6a554e9aa07cac4d4f0`.
- This maintains non-fallback version/tag reporting in build logs and runtime info.

### 3.2 SuSFS kernel patch scope for 4.19

- Imported and adapted SuSFS kernel-side 4.19 patchset into:
  - `fs/*` (path/mount/kstat/proc/overlay interfaces)
  - `include/linux/*` (`susfs.h`, `susfs_def.h`, `mount.h`, `sched.h`)
  - `kernel/*` (`kallsyms`, `sys`)
- Enabled `CONFIG_KSU_SUSFS=y` in:
  - `arch/arm64/configs/vendor/ext_config/nio-default.config`

### 3.3 Conflict resolution strategy

- Applied upstream SuSFS 4.19 patch and manually resolved rejects in tree-specific locations:
  - `fs/namespace.c`
  - `fs/overlayfs/readdir.c`
  - `fs/proc/task_mmu.c`
  - `include/linux/mount.h`
- Preserved existing Lineage/Kona semantics while applying SuSFS behavior hooks.

### 3.4 ABI and API compatibility strategy

KernelSU-Next `legacy-susfs` expected additional SUSFS symbols/commands not present in the imported 4.19 patchset.  
A compatibility layer was added in `fs/susfs.c` + `include/linux/susfs*.h`:

- Added missing command constants and `SUSFS_MAGIC`.
- Added missing symbol entry points used by KernelSU-Next legacy-susfs.
- Unified supercall-facing function signatures to `void __user *` where KernelSU passes generic pointers.
- Mapped try-umount flow to the symbol available in this tree (`try_umount`) to resolve final link failures.

## 4. Milestones and Commits

1. `f02dfd31a9ef`  
   `susfs: add KernelSU-Next legacy API compatibility layer`
2. `353107598df8`  
   `susfs: fix header visibility warnings for opaque structs`
3. `169f8455ee33`  
   `susfs: align userspace pointer APIs with KernelSU supercall ABI`
4. `36d9b30113ec`  
   `susfs: use KernelSU try_umount symbol for non-gki integration`

The current branch also includes the main legacy-susfs + SuSFS kernel integration delta (submodule pointer, fs/include/kernel/config changes).

## 5. Files/Areas Touched

- KernelSU submodule pointer (`KernelSU-Next`)
- Defconfig fragment:
  - `arch/arm64/configs/vendor/ext_config/nio-default.config`
- Kernel FS/namespace/proc/overlay hooks:
  - `fs/Makefile`, `fs/dcache.c`, `fs/namei.c`, `fs/namespace.c`,
    `fs/notify/fdinfo.c`, `fs/overlayfs/inode.c`, `fs/overlayfs/readdir.c`,
    `fs/overlayfs/super.c`, `fs/proc/cmdline.c`, `fs/proc/fd.c`,
    `fs/proc/task_mmu.c`, `fs/proc_namespace.c`, `fs/readdir.c`,
    `fs/stat.c`, `fs/statfs.c`
- New/updated SUSFS headers and source:
  - `fs/susfs.c`, `fs/sus_su.c`
  - `include/linux/susfs.h`, `include/linux/susfs_def.h`, `include/linux/sus_su.h`
- KABI/runtime structures:
  - `include/linux/mount.h`, `include/linux/sched.h`
- Ancillary kernel behavior:
  - `kernel/kallsyms.c`, `kernel/sys.c`

## 6. Validation Model

Per project workflow, build/flash/runtime validation is device-side and user-owned.  
Agent-side validation focused on log-driven compile/link failure triage and deterministic source fixes.

## 7. Operational Notes

After sync or clean checkout:

1. `cd kernel/motorola/sm8250`
2. `git submodule sync --recursive`
3. `git submodule update --init --recursive`
4. Ensure submodule HEAD is commit `4cc162e0...` on `legacy-susfs`.

## 8. Rollback Guidance

To revert legacy-susfs/susfs layer while keeping baseline KernelSU-Next integration:

1. Revert newest compatibility commits in reverse order:
   - `36d9b30113ec`
   - `169f8455ee33`
   - `353107598df8`
   - `f02dfd31a9ef`
2. Revert the main legacy-susfs+SuSFS integration commit(s) on this branch.
3. Point submodule back to prior `v3.2.0-legacy` commit if needed.

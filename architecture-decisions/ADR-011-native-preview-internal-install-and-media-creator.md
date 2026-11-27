# ADR-011: Native Interactive UI Preview, OSS Dev Infrastructure, Internal SSD Installation & SBM Manager Strategy

- **Status:** Accepted (Foundational Direction for SBM 0.0.1–0.0.3+; Extended by [ADR-012](ADR-012-profiles-network-recovery-modularity-and-ux.md))
- **Date:** 2026-11-26
- **Related ADRs:** [ADR-002](ADR-002-graphics-and-gui-separation.md), [ADR-006](ADR-006-storage-safety-and-tool-wrapping.md), [ADR-008](ADR-008-testing-virtualization-and-benchmark-methodology.md), [ADR-009](ADR-009-dual-architecture-offline-preview-media.md), [ADR-010](ADR-010-open-source-reuse-and-engineering-principles.md), [ADR-012](ADR-012-profiles-network-recovery-modularity-and-ux.md)
- **Supporting Research:**
  - `research/ecosystem/dev-infrastructure-and-testing-landscape-2026-11-26.md`
  - `research/validation/evidence/sbm-dev-resource-audit-2026-11-26.md`
  - `research/ecosystem/media-creator-evaluation.md`
  - `research/architecture/internal-install-and-calamares-evaluation.md`
  - `research/architecture/network-recovery-and-modularity-evaluation.md`

## Context

Following Wave 2 integration testing and host resource audits on a 16 GiB Apple Silicon workstation, Smart Boot Menu's product and engineering direction crystallized around six priorities (further extended by [ADR-012](ADR-012-profiles-network-recovery-modularity-and-ux.md) for configurable installation profiles, network recovery, 0SFS-style modularity, **`SBM Manager`** naming, and first-principles UX):

- **Priority A:** Native interactive SBM development using the actual UEFI renderer (`sbm-gfx`/`sbm-boot`) and Iced Core UI (`sbm-ui`) against simulated platform interfaces—serving as the **primary UX evaluation and refinement environment** ([ADR-012](ADR-012-profiles-network-recovery-modularity-and-ux.md)).
- **Priority B:** Efficient development infrastructure built primarily from existing open-source tools (per `ADR-010`).
- **Priority C (SBM 0.0.1):** Independently qualified architecture-specific bootable recovery media (`x86_64`, `aarch64`), real UEFI-to-Core workflow, GParted, and reboot.
- **Priority D (SBM 0.0.2):** Persistent internal SSD installation alongside existing operating systems into available disk space (supporting both `< 1 MiB` minimal `efi-only` and offline/cached internal Core profiles), firmware boot registration, uninstall, and recovery.
- **Priority E (SBM 0.0.3):** Simple cross-platform **`SBM Manager`** MVP (formerly called "Media Creator"), guided partition resizing for explicitly supported configurations, and initial signed `systemd-sysext` modularity, delivering the complete end-user flow:
  `Download SBM Manager -> configure profile & flash USB -> boot target computer -> Install SBM -> inspect and optionally adjust partitions -> confirm -> install -> reboot into SBM from internal SSD`.
- **Priority F (SBM 0.1+):** On-demand network recovery & ephemeral remote repair, full `systemd-sysext`/`confext` modular lifecycle, advanced OS installation (`sbm-catalog`), optional self-hosted remote management (`sbm-server`), production Secure Boot, and Apple Silicon physical support (`sbm-as`).

## Decision

### 1. Native Interactive Development Preview (Priority A — Track A)

Enable fast, sub-second native host UI and workflow iteration on macOS and Linux **without** launching VMs or duplicating UI codebases (engineering targets: **`< 1 s`** preview startup, **`< 5 s`** warm UI iteration):

- **Actual UEFI Renderer (`sbm-uefi`):** Expose an interactive native host runner built directly on the real `sbm-gfx` software renderer and `sbm-boot` discovery/policy/state crates against a simulated platform layer (`SimulatedEsp`, synthetic discovered targets, simulated `Clock`, simulated `BootAction`/handoff). Developers interact with real keyboard/mouse navigation, HiDPI scaling, themes, diagnostics, error overlays, and **Manage Computer** handoff using the exact rendering and policy code shipped to firmware.
- **Actual Iced Core UI (`sbm-core`):** Abstract `sbm-ui`'s daemon boundary (`sbm_runtime::ipc::transact`) behind a pluggable backend trait so the exact Iced `sbm-ui` binary runs natively on macOS/Linux against a deterministic `SimulatedCoreBackend` (including simulated disks, OS detections, recovery tools, and the 0.0.2/0.0.3 **Install SBM** wizard + simulated dry-run diffs).
- **Simulated End-to-End Flow:** Support transitioning from the interactive native UEFI preview (**Manage Computer**) into the interactive native Iced Core UI (passing a real Spec 03 `EfiHandoffPayload` fixture) and back on simulated reboot.
- **Primary UX Workbench & Strict Non-Qualification Rule:** Native interactive previews are the primary environment for evaluating and refining SBM's first-principles November 2026 UX ([ADR-012](ADR-012-profiles-network-recovery-modularity-and-ux.md)), and belong strictly to **Tier 1 (Simulated Testing)**. They MUST be visibly badged as simulated/mock mode and NEVER count as firmware, kernel, or storage qualification evidence.

### 2. Composed Open-Source Developer Infrastructure (Priority B — Track B)

Per `ADR-010` and Agent 16's build-vs-reuse audit:

- **Reuse directly:** Cargo 1.100 per-workspace `build.build-dir` + `mozilla/sccache` (`SCCACHE_ALLOW_INCREMENTAL=1`), `mkosi --incremental=yes` + `Format=sysext` + `systemd-sysext`, APFS `cp -c` / `qemu-img` overlays / `discard=unmap` + `fstrim`, and a thin root `justfile` (`just`) rather than a custom `sbm-dev` CLI or custom storage GC daemon.
- **Evaluate & adopt mature OSS runners:** Trial **Lima (`vmType: krunkit`, `libkrun >= v1.19.8`)** to replace custom QEMU cloud-init/`rsync` builder scripts (`sbm-lab/src/builder.rs` + `sbm-core/scripts/*-builder.sh`); adopt **U-Boot `sandbox`** on Linux CI for `sbm-as` EFI tests, **`os-autoinst` (`isotovideo`)** for Linux CI graphical/needle integration qualification, and **`labgrid`** when physical hardware / Apple Silicon board testing begins.
- **Consolidate & shrink `sbm-lab`:** Remove duplicated GPT/FAT32 writers (`gpt.rs`, `fatimg.rs`) in favor of `install/crates/sbm-media`; retain `sbm-lab` specifically for its genuine non-OSS gaps: the macOS multi-agent host RAM-admission guard (`resources.rs`) and thin macOS HVF UEFI smoke runner.

### 3. Internal SSD Installation & Guided Partition Resizing (SBM 0.0.2 & 0.0.3)

- **Preserve SBM's Rust Planner & Security Boundaries:** Keep `sbm-plan` / `sbm-exec` / `sbm-common` typed contracts (`DiskIdentity`, `PartitionProtectionClass`, `StorageOperationPlan`, `expected_partition_table_sha256` optimistic lock, GPT backup, install receipt, and explicit confirmation) as the sole planning and safety authority.
- **Do NOT Adopt Full Calamares Framework:** Per `research/architecture/internal-install-and-calamares-evaluation.md`, Calamares (`3.4.3`, Qt6/KF6/KPMcore/Python, `GPL-3.0-or-later`) would bloat `sbm-core` with a third GUI stack and does not model SBM's `dm-verity` recovery slots, non-destructive ESP coexistence, or uninstall/NVRAM restoration. Instead, adopt Calamares' proven safety patterns (strict deferred `show` -> `exec` execution, automount quiescing, visual "install alongside" partition bar UX, and ESP-flag hygiene).
- **Reuse Mature Partitioning & Filesystem Tools (`ADR-006`):** Invoke `lsblk`, `blkid`, `sgdisk`, `sfdisk`, `partprobe`, `dosfstools`, `efibootmgr`, `e2fsprogs` (`e2fsck`, `resize2fs`), `ntfsresize`, `btrfs-progs`, and `cryptsetup` strictly via direct `execve` subprocesses (preserving `MIT OR Apache-2.0` licensing for all SBM Rust crates). Keep bundled **GParted** as the manual escape hatch for unsupported layouts.
- **Staged Risk Isolation:**
  - **SBM 0.0.2:** Persistent internal SSD install **into existing unallocated space** (or explicit empty disk) only — zero destructive partition shrinking — supporting both **`efi-only` (`< 1 MiB`, independently useful and upgradeable without reinstalling the boot manager)** and **offline/cached internal Core**, plus `efibootmgr` boot registration, non-destructive ESP coexistence, clean **Uninstall** (restoring prior `BootOrder` and removing SBM files/partitions), and **Boot Repair / Recovery**.
  - **SBM 0.0.3:** Guided partition resizing **strictly allowlisted** to clean, unencrypted `ext4`, clean non-hibernated/non-BitLocker `NTFS`, and single-device `Btrfs` using the Golden Rule of Safe Shrinking (shrink FS with margin -> shrink partition keeping start sector -> expand FS to boundary). Hard-block BitLocker, LUKS, LVM/RAID/ZFS/XFS, dirty/hibernated NTFS, MBR disks, and Apple APFS.

### 4. Cross-Platform `SBM Manager` (Reuse First Before Custom Build)

Per `research/ecosystem/media-creator-evaluation.md`, `ADR-010`, and [ADR-012](ADR-012-profiles-network-recovery-modularity-and-ux.md):

- **Product Name:** The user-facing desktop provisioning and configuration application is **`SBM Manager`** (replacing "SBM Media Creator").
- **SBM 0.0.1 & 0.0.2:** Publish a schema-validated **`sbm-os-list.json`** compatible with **Raspberry Pi Imager (`>= 2.0.11`, `--repo` / App Options -> Content Repository)** alongside verified instructions for **USBImager**, **Fedora Media Writer**, **GNOME Impression**, and **Rufus**. Write zero custom GUI flasher code in 0.0.1–0.0.2.
- **SBM 0.0.3+ (`SBM Manager`):** Because `SBM Manager` must grow beyond simple raw-image flashing to configure recovery profiles (`efi-only`, `net-ondemand`, `ram-only`, `cached-usb`/`cached-ssd`, `offline-full`, `modular-custom`), preconfigure network/cache policies, select optional `systemd-sysext` modules, and mint revocable ephemeral remote-recovery grants (0.1+), the 0.0.3 architecture prototype explicitly evaluates two paths before committing to long-term implementation:
  1. A **minimal Apache-2.0 branded build/fork of `raspberrypi/rpi-imager`** (fastest path for pure cross-platform USB flashing, elevation, streaming decompression, and read-back verification, if SBM provisioning flows fit cleanly without fighting `cloudinit-rpi` assumptions).
  2. A **thin native Rust/Iced `SBM Manager`** sharing SBM's first-principles design system, `sbm-common` contracts, and TUF/profile/module logic while reusing mature OS flashing backends (`udisks2` / macOS DiskArbitration / Win32 raw-disk helpers) rather than writing a low-level block writer from scratch.

### 5. Mandatory Separation of Three Testing Tiers

All specifications, CI pipelines, and release gates MUST strictly separate three testing tiers (detailed in [docs/testing/benchmarks.md](../testing/benchmarks.md)):

1. **Tier 1 — Simulated Installation & Native UI Testing:** In-memory / regular-file / native host preview (`sbm-plan`, `sbm-exec` on regular files, interactive `sbm-gfx`/`sbm-ui` simulation). Fast (milliseconds–seconds), zero root/VM cost, never modifies block devices.
2. **Tier 2 — Disposable VM Disk Testing (`sbm-lab`):** Real firmware (OVMF/AAVMF/U-Boot), real Linux kernel + CLI storage tools, and real `efibootmgr` NVRAM mutations executing exclusively against **disposable copy-on-write QCOW2 overlays** of multi-OS disk fixtures (Windows/Linux/free-space/BitLocker/corrupt GPT). Mandatory gate for 0.0.1, 0.0.2, and 0.0.3.
3. **Tier 3 — Explicitly Authorized Physical Hardware Testing:** Opt-in manual testing on dedicated non-workstation test hardware (e.g., `EXP-HW-HP01`) with explicit human confirmation and recorded hardware provenance. **NEVER** automated, **NEVER** run against the developer's workstation disks, firmware `BootOrder`, or macOS `LocalPolicy`/SIP.

## Consequences

- **Positive:** Sub-second UI/UX iteration for both UEFI and Core via Track A; dramatically lower workstation RAM/SSD pressure via Track B and per-arch 0.0.1 media; clear risk separation between 0.0.2 (free-space internal install + `< 1 MiB` `efi-only` option) and 0.0.3 (guided shrinking + `SBM Manager` MVP + initial `sysext` modularity); zero wasted effort reimplementing mature flashers or filesystem tools.
- **Trade-offs:** Requires disciplined trait/backend boundaries in `sbm-uefi` and `sbm-core` so simulated previews never drift from real runtime behavior, and requires empirical comparison of a minimal `rpi-imager` fork vs. a thin Rust/Iced `SBM Manager` reusing OS flashing backends before locking the 0.0.3 implementation.

# Spec 04: Disk Identification, Storage Plans & Installation Plans (`v1`)

**Status:** Normative Draft v1.0 (`sbm-types`) — extended for SBM 0.0.2 internal SSD installation and SBM 0.0.3 guided partition resizing (see [ADR-006](../architecture-decisions/ADR-006-storage-safety-and-tool-wrapping.md) and [ADR-011](../architecture-decisions/ADR-011-native-preview-internal-install-and-media-creator.md)).

## 10. Stable Disk & Partition Identification (`DiskIdentity`, `PartitionIdentity`)

**Normative Rule:** No storage operation plan may target a disk solely by Linux kernel device node (`/dev/sda`, `/dev/nvme0n1`), because enumeration order is non-deterministic across reboots and USB insertions.

### `DiskIdentity`
- `stable_id: String` — canonical composite key (priority: `wwn:<wwn>` > `eui64:<eui64>` > `gpt:<disk_guid>` > `model-serial:<model>:<serial>`)
- `gpt_disk_guid: Option<String>`
- `wwn_or_eui64: Option<String>`
- `model: String`
- `serial_hash: Option<String>` (salted/hashed when reported remotely to avoid unnecessary PII leakage)
- `size_bytes: u64`
- `logical_sector_size: u32`
- `bus: DiskBus` (`Nvme | Sata | Usb | Virtio | Scsi | Mmc | Unknown`)
- `removable: bool`
- `read_only: bool`

### `PartitionProtectionClass`
Every discovered partition is classified automatically before any plan is accepted:
- `Normal` — standard user/OS partition
- `ProtectedEsp` — active EFI System Partition hosting SBM or existing OS bootloaders
- `ProtectedAppleSystem` — Apple `iSCPreboot`, `RecoveryOS`, macOS APFS System/Preboot/Recovery containers (**hard-blocked by default!**)
- `ProtectedEncrypted` — detected BitLocker (`bitlk`) or LUKS2 partition (read-only unless explicitly unlocked & confirmed; **hard-blocked for 0.0.3 guided resizing**)
- `ProtectedSbmRecovery` — currently booted `sbm-core` partition

## 11. Storage Operation Plans (`StorageOperationPlan`)

Every mutating disk operation in `sbm-core` and `install` MUST be compiled into a validated, immutable `StorageOperationPlan` before execution:
- `schema_version: "1.0"`
- `plan_id: String` (UUIDv7)
- `target_disk: DiskIdentity` (MUST match live disk byte-for-byte at execution time!)
- `expected_partition_table_sha256: String` (optimistic concurrency lock against concurrent changes)
- `operations: Vec<StorageOp>` (ordered sequence: `CreatePartition`, `ResizePartition`, `FormatFilesystem`, `DeletePartition`, `WriteImage`, `ClonePartition`, `SetPartitionFlags`)
- `destructive: bool` (auto-computed; `true` if ANY step deletes, overwrites, formats, or shrinks data)
- `reversible: bool` (MUST honestly be `false` for formats/deletes/resizes/clones!)
- `touched_protected_classes: Vec<PartitionProtectionClass>` (non-empty list triggers hard rejection unless an explicit multi-step override policy permits it)
- `dry_run_summary: Vec<String>` (human-readable lines presented to the operator/user before confirmation)

> **Planned 0.0.2 Schema Alignment ([`sbm-common#2`](https://github.com/smart-boot-menu/sbm-common/issues/2)):** Before SBM 0.0.2 internal SSD installation ships, extend `StorageOp::CreatePartition` with optional `unique_guid` and `attributes` fields and align `StorageOp::SetPartitionFlags` across `sbm-types`, `sbm-schema`, and `install` so DPS-compliant `dm-verity` root/hash partitions can be expressed cleanly in `StorageOperationPlan`.

## 11a. SBM 0.0.2 — Persistent Internal SSD Installation, Uninstall & Recovery

Installing Smart Boot Menu onto an internal GPT SSD alongside existing operating systems MUST preserve all SBM safety boundaries while reusing mature partitioning, filesystem, and UEFI tools (`sgdisk`, `sfdisk`, `dosfstools`, `efibootmgr`) as isolated `execve` subprocesses:

1. **Available-Space-Only Policy (No Shrinking in 0.0.2):**
   - SBM 0.0.2 MUST ONLY allocate SBM recovery partitions (`root-<arch>` + `root-<arch>-verity` matching the target machine's native architecture) inside **existing unallocated GPT space** (or an explicitly selected empty GPT disk).
   - SBM 0.0.2 MUST NEVER shrink, reformat, or overwrite an existing OS partition. If insufficient unallocated space exists, the UI informs the user and directs them to prepare free space (via GParted or their existing OS) or wait for SBM 0.0.3 guided resizing.
2. **Non-Destructive ESP Coexistence:**
   - **Preferred when existing ESP has sufficient verified free space (>= 192 MiB free after install):** Install SBM exclusively under `\EFI\sbm\` (`\EFI\sbm\sbm-<arch>.efi`, `\EFI\sbm\recovery\slot-a\sbm-core-<arch>.efi`, `\EFI\sbm\recovery\manifest.json`, `\EFI\sbm\install-receipt.json`). MUST NOT modify, overwrite, or delete `\EFI\Microsoft\`, `\EFI\BOOT\`, or any existing Linux/BSD bootloader directories.
   - **Fallback when existing ESP lacks free space (or no ESP exists):** Create a dedicated FAT32 SBM ESP (`>= 256 MiB`, formatted via `mkfs.fat -F 32`) in unallocated GPT space without touching the existing OS ESP.
3. **Pre-Flight Safety, Quiescing & Backup:**
   - Reject disks containing `ProtectedAppleSystem` partitions or non-GPT (MBR) partition tables.
   - Detect BitLocker (`bitlk`) partitions anywhere on the system; if present, require explicit user acknowledgement that they hold their **BitLocker Recovery Key** before altering UEFI `BootOrder`.
   - Before any write: disable automounting for target disk, verify target ranges are unmounted, re-verify live `DiskIdentity` and `expected_partition_table_sha256`, and capture a binary GPT backup (`sgdisk --backup`) plus current `efibootmgr -v` snapshot.
4. **Firmware Boot Registration & Install Receipt:**
   - Register a distinct UEFI boot entry via `efibootmgr -c -d <disk> -p <esp_part> -L "Smart Boot Menu" -l '\EFI\sbm\sbm-<arch>.efi'` and place it first in `BootOrder`.
   - Persist and `fsync` `\EFI\sbm\install-receipt.json` recording: SBM version, `target_disk.stable_id`, ESP PARTUUID, created recovery partition PARTUUIDs (data + verity), installed file SHA-256s, prior `BootOrder`, and newly created `BootXXXX` ID.
5. **Clean Uninstall & Boot Recovery:**
   - **Uninstall:** Reads `\EFI\sbm\install-receipt.json`, restores the recorded prior `BootOrder` (filtering out deleted entries), deletes SBM's `BootXXXX` NVRAM entry (`efibootmgr -b XXXX -B`), removes `\EFI\sbm\` (leaving all other OS loaders untouched), and—when explicitly confirmed—deletes only the exact PARTUUIDs recorded as created by SBM in the install receipt.
   - **Boot Recovery / Repair:** When a Windows Update or firmware reset overwrites `BootOrder`, booting SBM recovery USB provides a non-destructive **Repair SBM Boot Registration** action that verifies `\EFI\sbm\sbm-<arch>.efi` on the internal ESP and re-creates/re-prioritizes its `efibootmgr` entry without modifying partitions.

## 11b. SBM 0.0.3 — Guided Partition Resizing (Explicitly Supported Configurations Only)

When an internal GPT SSD has no sufficient unallocated space, SBM 0.0.3 MAY offer a guided partition shrink step ahead of the §11a installation pipeline, subject to strict allowlisting and the **Golden Rule of Safe Shrinking**:

1. **Explicitly Supported Configurations (Allowlist):**
   - Clean, unencrypted **`ext4`** (`e2fsck -f` + offline `resize2fs` + `sfdisk`/`sgdisk` + `resize2fs` final fit).
   - Clean, unencrypted **`NTFS`** with **Windows Fast Startup / hibernation disabled** (no dirty flag / `hiberfil.sys`) and **BitLocker disabled** (`ntfsresize --check --info --no-action` + `ntfsresize --size` + `sfdisk`/`sgdisk` + `ntfsresize` final fit; display post-resize `chkdsk` notice).
   - Clean, unencrypted, single-device **`Btrfs`** with verified unallocated headroom (`btrfs filesystem usage` + mounted `btrfs filesystem resize` + unmount + `sfdisk`/`sgdisk` + remount `btrfs filesystem resize max`).
2. **Explicitly Hard-Blocked Configurations (Must Refuse Guided Resize):**
   - BitLocker / Windows Device Encryption (`ProtectedEncrypted`), LUKS/dm-crypt, LVM, MDADM software RAID, ZFS, Windows Dynamic Disks / Storage Spaces, XFS (non-shrinkable), dirty or hibernated NTFS, `ProtectedAppleSystem` (APFS), `ProtectedEsp`, `ProtectedSbmRecovery`, and MBR disks.
   - Refusals MUST explain *why* the partition is blocked and provide actionable guidance (e.g., disable Windows Fast Startup, back up BitLocker recovery key and shrink in Windows Disk Management, or use bundled GParted manually).
3. **Mandatory Safe-Shrink Execution Sequence:**
   - Verify pre-flight consistency and minimum-size probes with zero writes.
   - Present exact before/after visual partition map, dry-run command summary, and honest **non-reversibility / backup disclosure** (`destructive: true, reversible: false`).
   - Shrink **filesystem first** with a safety margin (`>= 64 MiB` below target partition end to prevent SI/IEC rounding truncation) -> shrink **GPT partition end sector only** (keeping identical start sector, 1 MiB aligned) -> run `partprobe`/`udevadm settle` -> **expand filesystem** to exact new partition boundary -> re-verify metadata before creating SBM partitions.

## 12. OS Installation Plans (`OsInstallPlan`)

Combines a verified catalog entry (`Spec 05`) + validated `StorageOperationPlan` + post-install configuration (SBM 0.1+):
- `plan_id: String`
- `catalog_entry_id: String` & `catalog_release_version: String`
- `target_arch: CpuArch`
- `strategy: InstallStrategy` (`DirectInstallerBoot | RootfsImageExtract | BootcOci | WindowsWimApply | BsdImage`)
- `artifact_verification: Vec<VerifiedArtifactRef>` (exact URL/local path + SHA-256 + TUF/signature reference)
- `storage_plan: StorageOperationPlan`
- `boot_registration: BootRegistrationPlan` (whether to create BLS entry, EFI loader, or SBM catalog target, and whether to set as default or keep current default)
- `offline_media_only: bool`

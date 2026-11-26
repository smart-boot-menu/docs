# Spec 04: Disk Identification, Storage Plans & Installation Plans (`v1`)

**Status:** Normative Draft v1.0 (`sbm-types`)

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
- `ProtectedEncrypted` — detected BitLocker (`bitlk`) or LUKS2 partition (read-only unless explicitly unlocked & confirmed)
- `ProtectedSbmRecovery` — currently booted `sbm-core` partition

## 11. Storage Operation Plans (`StorageOperationPlan`)

Every mutating disk operation in `sbm-core` MUST be compiled into a validated, immutable `StorageOperationPlan` before execution:
- `schema_version: "1.0"`
- `plan_id: String` (UUIDv7)
- `target_disk: DiskIdentity` (MUST match live disk byte-for-byte at execution time!)
- `expected_partition_table_sha256: String` (optimistic concurrency lock against concurrent changes)
- `operations: Vec<StorageOp>` (ordered sequence: `CreatePartition`, `ResizePartition`, `FormatFilesystem`, `DeletePartition`, `WriteImage`, `ClonePartition`, `SetPartitionFlags`)
- `destructive: bool` (auto-computed; `true` if ANY step deletes, overwrites, formats, or shrinks data)
- `reversible: bool` (MUST honestly be `false` for formats/deletes/resizes/clones!)
- `touched_protected_classes: Vec<PartitionProtectionClass>` (non-empty list triggers hard rejection unless an explicit multi-step override policy permits it)
- `dry_run_summary: Vec<String>` (human-readable lines presented to the operator/user before confirmation)

## 12. OS Installation Plans (`OsInstallPlan`)

Combines a verified catalog entry (`Spec 05`) + validated `StorageOperationPlan` + post-install configuration:
- `plan_id: String`
- `catalog_entry_id: String` & `catalog_release_version: String`
- `target_arch: CpuArch`
- `strategy: InstallStrategy` (`DirectInstallerBoot | RootfsImageExtract | BootcOci | WindowsWimApply | BsdImage`)
- `artifact_verification: Vec<VerifiedArtifactRef>` (exact URL/local path + SHA-256 + TUF/signature reference)
- `storage_plan: StorageOperationPlan`
- `boot_registration: BootRegistrationPlan` (whether to create BLS entry, EFI loader, or SBM catalog target, and whether to set as default or keep current default)
- `offline_media_only: bool`

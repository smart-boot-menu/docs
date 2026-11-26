# ADR-006: Storage Safety Model & Wrapping Mature Filesystem Utilities

- **Status:** Accepted (Foundational)
- **Date:** 2026-11-26
- **Supporting Research:** `research/ecosystem/recovery-and-storage.md`, `research/ecosystem/os-provisioning.md`

## Context

Storage operations (partitioning, resizing, formatting, cloning, OS installation, decryption) are the highest-risk operations SBM performs. A single bug can permanently destroy user data or render a machine unbootable.

## Decision

1. **Do NOT Rewrite Mature Low-Level Storage Tools in Rust:** Wrap battle-tested upstream utilities (`util-linux` `lsblk`/`blkid`/`sfdisk`, `gptfdisk` `sgdisk`, `e2fsprogs`, `dosfstools`, `ntfs-3g`/`ntfsprogs`, `btrfs-progs`, `exfatprogs`, `partclone`, `cryptsetup 2.8.9`, `ddrescue`, `testdisk`, `wimlib-imagex 1.14.5`, and interactive `gparted 1.8.1`) via direct `execve` / `Command::args` (NEVER shell interpolation).
2. **Implement Safety Orchestration in Rust (`sbm-core` + `sbm-common`):**
   - **Read-only discovery by default:** All initial probes open devices read-only.
   - **Stable Disk Identity:** Never address disks by unstable `/dev/sdX` paths in plans; require matching composite `DiskIdentity` (GPT Disk GUID + Partition GUID + WWN/EUI-64/Serial + exact byte size). Reject execution if the disk changed between planning and execution.
   - **Explicit `StorageOperationPlan`:** Every mutation must be validated into a typed plan (`sbm-common`), checked against **Platform-Critical Partition Protection Rules** (active ESP, Apple `iSCPreboot`/`Recovery`, BitLocker-encrypted volumes, active recovery slots), reviewed with a human-readable dry-run diff, and confirmed with a distinct destructive authorization token / user confirmation.
   - **Honest Non-Reversibility Disclosure:** UI and APIs MUST explicitly state that operations like formatting, shrinking filesystems, overwriting partitions, or interrupted resizes are **NOT transactional or reversible** — backup before destructive changes is recommended and enforced in UX warnings.
   - **Licensing Isolation:** Invoking GPL-2.0/3.0 CLI tools as isolated subprocesses preserves SBM's `MIT OR Apache-2.0` licensing for all original Rust code.

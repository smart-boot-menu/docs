# ADR-009 — Dual-architecture offline preview media

**Status** Accepted design for 0.0.1. VM qualification is pending.

The first integration artifact is one removable GPT image qualified byte-for-byte on x86_64 OVMF and AArch64 AAVMF. A shared ESP contains native EFI entry points and recovery UKIs. Keep mkosi's native verity pairs in the image. This allows each kernel to select its intended immutable root without a partition scanner or a second image-building stack.

UEFI starts the UKI via its device path with no external options. Keep static root and verity settings in the UKI. Pass boot-specific data through the identified ESP and volatile variables. Use the existing attempt and success records to link the GUI and next firmware boot. FAT writes require observable error handling and flushes, with interrupted state treated as unconfirmed.

The component responsibilities follow the existing repository boundaries. Core builds each native recovery input, UEFI builds the menus, Installer assembles regular files, Lab qualifies the unchanged candidate, and the coordination repository records evidence and release status.

An ordinary offline boot gates the preview. Enrolled-key Secure Boot requires its own positive and negative evidence. Physical Apple Silicon, remote recovery and disk mutations have separate qualification tracks.

See [Spec 02](../specifications/02-boot-policy-and-commands.md), [Spec 03](../specifications/03-efi-linux-handoff-and-recovery.md), and [Spec 07](../specifications/07-releases-compatibility-and-diagnostics.md).

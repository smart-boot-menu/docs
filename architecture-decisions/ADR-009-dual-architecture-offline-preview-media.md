# ADR-009 — Dual-architecture offline preview media

**Status** Accepted design for 0.0.1 (`sbm-0.0.1-uefi-dualarch.raw` verified in VM lab; physical hardware pending). **Amended (2026-11-26, pending Agent 11 review):** architecture-specific distributions are **recommended**; the universal dual-architecture image is **experimental and optional**; local development builds default to native architecture only.

## Distribution Policy (2026-11-26 Amendment)

- **Recommended distributions:** Architecture-specific single-architecture images (`x86_64` and generic `aarch64`) are the recommended user and CI distributions. Single-architecture media halves download size (~1.3 GiB vs ~2.2 GiB raw), avoids cross-architecture emulation during local builds, and keeps firmware startup free of foreign-architecture artifacts.
- **Experimental / optional universal image:** The combined dual-architecture image (`sbm-<version>-uefi-dualarch.raw[.zst]`) remains supported as an **experimental, optional** convenience artifact for technicians carrying one multi-architecture USB drive. It is **not** required for ordinary development or default release lanes.
- **Apple Silicon:** Remains a separate experimental platform track (`sbm-as`, `kernel-16k` Fedora Asahi Minimal; see ADR-003 and ADR-005).
- **Local development default:** Local development builds (`sbm-uefi`, `sbm-core`, `just`) MUST default to the host's native architecture (`aarch64` on Apple Silicon, `x86_64` on x86-64 hosts) with compression disabled. Cross-architecture and dual-architecture builds are explicit opt-in operations reserved for integration/release qualification or native remote/CI builders.

## Original 0.0.1 Dual-Architecture Design (Preserved for Optional Universal Media)

The first integration artifact is one removable GPT image qualified byte-for-byte on x86_64 OVMF and AArch64 AAVMF. A shared ESP contains native EFI entry points and recovery UKIs. Keep mkosi's native verity pairs in the image. This allows each kernel to select its intended immutable root without a partition scanner or a second image-building stack.

UEFI starts the UKI via its device path with no external options. Keep static root and verity settings in the UKI. Pass boot-specific data through the identified ESP and volatile variables. Use the existing attempt and success records to link the GUI and next firmware boot. FAT writes require observable error handling and flushes, with interrupted state treated as unconfirmed.

The component responsibilities follow the existing repository boundaries. Core builds each native recovery input, UEFI builds the menus, Installer assembles regular files, Lab qualifies the unchanged candidate, and the coordination repository records evidence and release status.

An ordinary offline boot gates the preview. Enrolled-key Secure Boot requires its own positive and negative evidence. Physical Apple Silicon, remote recovery and disk mutations have separate qualification tracks.

See [Spec 02](../specifications/02-boot-policy-and-commands.md), [Spec 03](../specifications/03-efi-linux-handoff-and-recovery.md), and [Spec 07](../specifications/07-releases-compatibility-and-diagnostics.md).

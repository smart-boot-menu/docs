# Smart Boot Menu — Documentation (`docs`)

Canonical technical documentation, normative interface specifications, Architecture Decision Records (ADRs), threat models, hardware profiles, testing methodology, and architecture diagrams for [Smart Boot Menu](https://github.com/smart-boot-menu).

Modeled after dedicated documentation repositories (`0SFS/docs`), this repository is the **single source of truth** for architectural decisions and specifications across all Smart Boot Menu repositories:
- Primary-source ecosystem studies and raw empirical evidence live in [`smart-boot-menu/research`](https://github.com/smart-boot-menu/research).
- Executive project overview, roadmap, and agent launch prompts live in [`smart-boot-menu/sbm`](https://github.com/smart-boot-menu/sbm).
- Canonical shared Rust types, JSON schemas, and test vectors implementing these specs live in [`smart-boot-menu/sbm-common`](https://github.com/smart-boot-menu/sbm-common).

## Contents

### Architecture Decision Records (`architecture-decisions/`)
- [ADR-001: Two-Tier Execution Architecture (UEFI Fast Path + Linux Management Path)](architecture-decisions/ADR-001-two-tier-uefi-and-linux-architecture.md)
- [ADR-002: Graphics & GUI Strategy — Shared Design System, Separate Renderers](architecture-decisions/ADR-002-graphics-and-gui-separation.md)
- [ADR-003: `sbm-core` Minimal Linux OS — Unified `mkosi` (Fedora Minimal for x86/ARM64 PCs + Light Fedora Asahi Minimal for Apple Silicon)](architecture-decisions/ADR-003-sbm-core-minimal-linux-and-apple-silicon-asahi.md)
- [ADR-004: `sbm-uefi` Rust Stack, Opportunistic Networking & ESP-First Persistence](architecture-decisions/ADR-004-uefi-rust-stack-and-esp-persistence.md)
- [ADR-005: Apple Silicon Integration (`m1n1` + Asahi U-Boot + Shared `sbm-uefi`)](architecture-decisions/ADR-005-apple-silicon-m1n1-uboot-strategy.md)
- [ADR-006: Storage Safety Model & Wrapping Mature Filesystem Utilities](architecture-decisions/ADR-006-storage-safety-and-tool-wrapping.md)
- [ADR-007: Self-Hosted Server Stack, Cryptographic Identity & Remote Protocol](architecture-decisions/ADR-007-remote-management-server-and-security-protocol.md)
- [ADR-008: Virtualization, Emulation, Fault Injection & Benchmark Methodology](architecture-decisions/ADR-008-testing-virtualization-and-benchmark-methodology.md)
- [ADR-009: Architecture-Specific Offline Preview Media (With Optional Experimental Dual-Arch)](architecture-decisions/ADR-009-dual-architecture-offline-preview-media.md)
- [ADR-010: Open-Source Reuse & Engineering Principles](architecture-decisions/ADR-010-open-source-reuse-and-engineering-principles.md)
- [ADR-011: Native Interactive UI Preview, OSS Dev Infrastructure, Internal SSD Installation & SBM Manager Strategy](architecture-decisions/ADR-011-native-preview-internal-install-and-media-creator.md)
- [ADR-012: Configurable Installation Profiles, Network Recovery, 0SFS-Style Modularity, SBM Manager Naming & First-Principles UX](architecture-decisions/ADR-012-profiles-network-recovery-modularity-and-ux.md)

### Versioned Specifications (`specifications/`)
- [Specifications Index & Versioning Policy](specifications/README.md)
- [01 — Identifiers & Platform Capabilities](specifications/01-identifiers-and-capabilities.md)
- [02 — Boot Policies, Commands & Attempt Reporting](specifications/02-boot-policy-and-commands.md)
- [03 — EFI-to-Linux Handoff & Recovery Discovery](specifications/03-efi-linux-handoff-and-recovery.md)
- [04 — Disk Identification, Storage Plans & Installation Plans](specifications/04-storage-and-installation-plans.md)
- [05 — Operating System Catalog Format & Trust](specifications/05-os-catalog.md)
- [06 — Remote Enrollment, AuthN/AuthZ & Signed Command Envelopes](specifications/06-remote-protocol-and-security.md)
- [07 — Release Manifests, Compatibility Negotiation & Diagnostics](specifications/07-releases-compatibility-and-diagnostics.md)

### Security, Hardware, Testing & Diagrams
- [Threat Model & Safety Invariants](threat-model/README.md)
- [Target Hardware Profiles & Experiments](hardware/target-hardware.md)
- [Apple Silicon Boot Architecture](hardware/apple-silicon-boot-architecture.md)
- [Engineering Targets, Testing Tiers & Benchmark Methodology](testing/benchmarks.md)
- [Tiered Development Workflow & Quick-Start (`just`)](testing/development-workflow.md)
- [System Architecture & Sequence Diagrams (Mermaid)](diagrams/architecture-diagrams.md)
- [Validation & Documentation Layout](validation/layout.md)

## Status Labels Used Across Documents

Every document distinguishes:
- **Requirement (Normative):** Mandatory behavior (`MUST` / `MUST NOT` per RFC 2119).
- **Validated Decision:** Accepted ADR backed by primary-source evidence or test validation.
- **Architectural Proposal:** Concrete design pending proof-of-concept validation.
- **Open Research Question:** Explicitly unverified area with a defined experiment to resolve it.

## License

Original documentation in this repository is dual-licensed under [MIT](LICENSE-MIT) OR [Apache-2.0](LICENSE-APACHE).

# ADR-012: Configurable Installation Profiles, Network Recovery, 0SFS-Style Modularity, SBM Manager Naming & First-Principles UX

- **Status:** Accepted (Owner Product & Architecture Direction — Staged Across SBM 0.0.1–0.1+)
- **Date:** 2026-11-26
- **Related ADRs:** [ADR-001](ADR-001-two-tier-uefi-and-linux-architecture.md), [ADR-002](ADR-002-graphics-and-gui-separation.md), [ADR-003](ADR-003-sbm-core-minimal-linux-and-apple-silicon-asahi.md), [ADR-007](ADR-007-remote-management-server-and-security-protocol.md), [ADR-009](ADR-009-dual-architecture-offline-preview-media.md), [ADR-010](ADR-010-open-source-reuse-and-engineering-principles.md), [ADR-011](ADR-011-native-preview-internal-install-and-media-creator.md)
- **Supporting Research:**
  - `research/architecture/network-recovery-and-modularity-evaluation.md`
  - `research/ecosystem/media-creator-evaluation.md`
  - `research/architecture/internal-install-and-calamares-evaluation.md`

## Context

Smart Boot Menu is designed to serve both local everyday multi-boot/recovery users and one of its primary product use cases: **remotely repairing a computer whose installed operating system is completely broken**, without forcing monolithic media, mandatory cloud services, or unauthorized touches to internal disks.

To align engineering execution with long-term product architecture without derailing the immediate SBM 0.0.1–0.0.3 milestones, this ADR ratifies six owner product and architecture decisions:

1. Independently configurable installation and execution profiles (from a `< 1 MiB` EFI boot manager up to full offline and modular deployments).
2. Standards-based on-demand network recovery and revocable, ephemeral remote repair sessions.
3. 0SFS-style maximum modularity built on existing `mkosi`, `systemd-sysext`/`systemd-confext`, signed `dm-verity` DDIs, and `systemd-sysupdate` (without inventing a general-purpose package manager).
4. Product naming: **`SBM Manager`** for the user-facing desktop provisioning and configuration application, retaining **`SBM Core`** as the internal management OS name for now.
5. First-principles November 2026 UX philosophy evaluated primarily in Agent 18's native interactive simulator (Track A).
6. Concrete initial engineering performance targets, strictly distinguished from measured capabilities.

---

## Decision

### 1. Independently Configurable Installation & Execution Profiles

SBM MUST support six independently configurable installation and execution profiles rather than coupling the UEFI boot manager to a single monolithic recovery image (normative layout and safety rules in [Spec 03](../specifications/03-efi-linux-handoff-and-recovery.md) §9a):

| Profile ID | Description | Persistent Storage Footprint | Allowed Internal SSD Writes |
|---|---|---|---|
| `efi-only` | Minimal UEFI boot manager only (`sbm-uefi`), targeting **`< 1 MiB`** EFI binary. Independently useful for OS discovery, fast boot selection, diagnostics, and future upgrade/profile expansion **without reinstalling the boot manager**. | `< 1 MiB` binary on ESP + small config/handoff metadata | Existing ESP only (explicitly installed); zero extra partitions |
| `net-ondemand` | Boot manager (or recovery USB) fetches and cryptographically authenticates `sbm-core` over the network on demand rather than shipping a large static payload. | Minimal EFI + optional compact signed Linux Network-Bootstrap UKI | **None** unless user explicitly enables `cached-ssd` |
| `cached-usb` / `cached-ssd` | Optional persistent caching of verified `sbm-core` images and optional extension modules on the recovery USB or an explicitly authorized internal SSD partition. | Authorized USB partition or authorized internal SBM partition | **NEVER** cache to an internal disk without explicit user authorization |
| `ram-only` | Fully RAM-resident execution of `sbm-core` and active extensions with zero persistent disk writes (ideal for forensic recovery, failing drives, or untrusted hosts). | Zero persistent writes after boot | **STRICTLY FORBIDDEN:** MUST NOT use the target internal SSD for temporary storage, cache, logs, or swap |
| `offline-full` | Self-contained offline installation containing `sbm-uefi`, verified `sbm-core` base, and profile-bundled recovery tools (`0.0.1` primary media & `0.0.2` offline internal SSD install). | Native-arch ESP + verified EROFS root + verity hash | Explicitly authorized target medium only |
| `modular-custom` | Minimal base + user-selected optional `systemd-sysext`/`systemd-confext` capability modules (downloaded, cached, activated, deactivated, or removed on demand). | Base (`< 500 MiB` compressed target) + selected extensions | Explicitly authorized cache medium only |

A minimal `efi-only` installation MUST remain independently useful and upgradeable to `net-ondemand`, `cached-ssd`, `offline-full`, or `modular-custom` purely by provisioning verified recovery manifests/images and optional extensions—without reinstalling or replacing the boot manager itself.

### 2. Standards-Based Network Recovery & Ephemeral Remote Repair

A primary SBM product use case is **remotely repairing a computer whose installed operating system is completely broken**:

> Working computer runs **`SBM Manager`** (optionally preconfiguring network settings, required modules, cache policy, and an explicitly authorized ephemeral remote-access grant) → flashes recovery USB → broken computer boots SBM from USB → downloads and authenticates `sbm-core` → executes in RAM (`ram-only`) or authorized cache (`cached-usb`/`cached-ssd`) → establishes an authenticated, encrypted, revocable remote session → owner repairs the broken OS.

Per `research/architecture/network-recovery-and-modularity-evaluation.md` and `ADR-010`:

- **Prefer Existing Standards (No Custom Network Boot Protocol):**
  1. **Standard UEFI HTTP(S) Boot (UEFI 2.5+)** as the zero-Linux wired Ethernet fast path for `efi-only` and enterprise/modern UEFI firmware (plus interoperability with standard **iPXE** boot environments where operators already deploy iPXE, without depending on iPXE's obsolete Wi-Fi drivers or GPL-2.0 codebase inside SBM).
  2. **Signed Minimal Linux Network-Bootstrap UKI + `dm-verity` `switch-root`** as the primary consumer-hardware network recovery path (bundled on a lightweight `SBM Manager` recovery USB or optional ESP). Because commodity laptops almost never ship UEFI Wi-Fi drivers or default firmware HTTPS CA bundles—and `kexec_file_load(2)` under Secure Boot lockdown cannot directly execute a UKI's `systemd-stub`—a compact signed Linux bootstrap UKI provides real Ethernet + consumer Wi-Fi drivers (`iwd`/`wpa_supplicant` + curated `linux-firmware`), full Linux TLS CA verification, HTTP `Range` resumable downloads, TUF/SHA-256 verification, and `switch-root` directly into the verified `dm-verity` EROFS root + signed extensions using the same signed kernel.
- **Resilience & Failure-Mode Requirements (Normative in [Spec 03](../specifications/03-efi-linux-handoff-and-recovery.md) §9a):**
  - Support both **Ethernet** and **consumer Wi-Fi** (optional preconfigured WPA2/WPA3 credentials stored strictly for the recovery medium, never copied to internal disks).
  - Detect **insufficient RAM** (`≤ 2 GiB`) before attempting `ram-only` full-image buffering: fail clearly or offer an authorized USB cache / reduced minimal recovery profile—**never** silently spill swap or temp files onto the target internal SSD.
  - Handle **unavailable networking**, captive portals, DNS/TLS errors, and **interrupted downloads** cleanly with HTTP `Range` resume, explicit diagnostics, and graceful fallback to local offline recovery when present.
- **Strict Ephemeral Remote-Access Security Boundary (Normative in [Spec 06](../specifications/06-remote-protocol-and-security.md) §16a):**
  - Remote access MUST be **opt-in, authenticated, encrypted, revocable, and limited to the explicitly authorized recovery session**.
  - **NEVER** store permanent shared credentials, reusable admin keys, or unattended arbitrary command-execution hooks on recovery USB drives.
  - Every remote-recovery USB carries at most a short-lived, single-session **Ephemeral Recovery Grant** (time-bounded TTL, pinned peer/relay public key, explicit allowed scopes), displays an unmistakable on-screen active-session indicator with a one-action local kill switch, and revokes automatically on exit, expiry, or reboot.

### 3. Maximum Modularity (0SFS-Style, Zero Custom Package Manager)

Following the 0SFS-style modularity philosophy:

- **Small Mandatory Base:** Keep the mandatory `sbm-core` base as small as practical (initial engineering target: **`< 500 MiB`** compressed per architecture). Pre-bundle components only when technically necessary for booting/recovery, justified by the selected installation profile, or explicitly selected by the user.
- **Reuse Existing Linux Immutability & Extension Standards (`ADR-010`):**
  - Build optional capabilities with **`mkosi`** (`Format=sysext` and `Format=confext`, `Overlay=yes`, `BaseTrees=`) as signed `dm-verity`-protected GPT Disk Discovery Images (DDIs).
  - Activate/deactivate at runtime using **`systemd-sysext`** and **`systemd-confext`** (`SYSEXT_LEVEL=`, `SYSEXT_SCOPE=`, `EXTENSION_RELOAD_MANAGER=1`), enforcing signed-only production images via `--image-policy='root=signed+absent:usr=signed+absent:=unused+absent'` and measuring initrd extensions into **TPM2 PCR 13** (keeping PCR 11 clean).
  - Deliver version-locked base/feature updates and independent module transfers via **`systemd-sysupdate`** (`Features=` / `--component=`) backed by signed TUF release metadata.
  - **Do NOT invent a general-purpose package manager.**
- **Full User Lifecycle & Metadata Contract (Normative in [Spec 07](../specifications/07-releases-compatibility-and-diagnostics.md) §17a):**
  - Users can **download, cache, activate, deactivate, and remove** optional capability modules while preserving immutable verified production roots.
  - Every module publishes signed metadata declaring `id`, `version`, `sysext_level` compatibility, `arch`, flat acyclic `depends_on`, integrity/signature digests (TUF + PKCS#7 `dm-verity`), download/disk/RAM requirements, and offline/cache availability.

### 4. Product & Component Naming

- **`SBM Manager`** is the official name of the user-facing desktop provisioning and configuration application (superseding "SBM Media Creator" across all roadmaps, specs, and issues). Over time, `SBM Manager` evolves from simple cross-platform USB flashing (0.0.3 MVP) to configuring recovery profiles, preconfiguring network/cache policies, managing installations, configuring revocable remote-recovery grants, and managing optional modules (0.1+).
- **`SBM Core`** is retained as the internal management OS name for now. User-facing naming may evolve independently without forcing crate or protocol renames.

### 5. First-Principles UX Philosophy (November 2026)

- Existing boot managers (GRUB, rEFInd, systemd-boot) and legacy recovery discs are useful technical references, **not** visual or architectural constraints.
- Prioritize **first-principles reasoning** and the best practical state-of-the-art UX technologies available in November 2026 rather than replicating historical conventions merely because they are familiar.
- Core UX principles across both `sbm-uefi` and `sbm-core`:
  - **Speed & responsiveness:** Sub-250 ms UEFI paint, responsive keyboard/mouse/touch-friendly layouts across 720p–4K HiDPI displays.
  - **Clarity & progressive disclosure:** Clean primary actions up front (boot targets, Manage Computer, Install/Repair), with deep technical telemetry, partition tables, logs, and advanced knobs revealed progressively on demand.
  - **Accessibility:** High-contrast legible typography, crisp focus indicators, keyboard-first navigation without hidden shortcuts, and deterministic layouts.
  - **Useful diagnostics & minimal resource consumption:** Actionable human-readable failure hints rather than raw error codes, without sacrificing low RAM/CPU overhead.
- **Primary Evaluation Environment:** **Agent 18's native interactive simulator (Track A)**—running the real `sbm-gfx`/`sbm-boot` UEFI renderer and real Iced `sbm-ui` natively against simulated platform interfaces—is the **primary environment** for evaluating, critiquing, and refining the actual SBM UI (`< 1 s` preview startup, `< 5 s` warm UI iteration).

### 6. Initial Engineering Performance Targets (Aspirational vs. Measured)

The following initial engineering targets are adopted as design goals in [docs/testing/benchmarks.md](../testing/benchmarks.md), subject to empirical measurement and refinement. Every document and release report MUST strictly distinguish these **aspirational targets** from **measured capabilities**:

- UEFI initialization: **`< 250 ms`**
- Minimal EFI application (`efi-only` / `sbm-uefi`): **`< 1 MiB`**
- Compressed base management OS (`sbm-core` base): **`< 500 MiB`**
- Local recovery boot on modern hardware: **`< 10 s`**
- Basic recovery on low-memory hardware: **`≤ 2 GiB RAM`** where practical
- Native interactive UI preview startup (Track A): **`< 1 s`**
- Warm UI iteration (Track A edit-compile-preview): **`< 5 s`**
- Zero mandatory internet, account, or cloud service
- Zero unauthorized physical disk modifications
- Existing dual-architecture combined image preserved strictly as **optional and experimental** (non-gating, per [ADR-009](ADR-009-dual-architecture-offline-preview-media.md))

---

## Incremental Delivery & Non-Disruption Sequencing

These capabilities MUST be delivered incrementally without destabilizing active engineering tracks (Agents 13, 16, 18):

1. **Tracks A & B (Continuous Parallel):** Agent 18's native interactive simulator (primary UX refinement workbench) + Agent 16's OSS developer infrastructure.
2. **SBM 0.0.1:** Independently qualified per-arch offline recovery media (`x86_64`, `aarch64`; dual-arch optional experimental), UEFI → Core handoff, GParted, reboot, and zero-code `sbm-os-list.json`.
3. **SBM 0.0.2:** Persistent internal SSD installation into available space—supporting both **`efi-only` (`< 1 MiB`, independently useful & upgradeable)** and **offline/cached internal Core**—plus `efibootmgr` registration, uninstall, and recovery.
4. **SBM 0.0.3:** **`SBM Manager` MVP** (profile config, USB flashing, cache/module selection), allowlisted guided partition resizing, and **initial signed `systemd-sysext` modularity** (splitting optional tools out of the `< 500 MiB` compressed base).
5. **SBM 0.1:** **On-Demand Network Recovery & Ephemeral Remote Repair** (`SBM Manager` recovery USB preconfiguration, UEFI HTTP(S) Boot + signed minimal Linux Network-Bootstrap UKI, `ram-only` zero-SSD-touch mode, authorized caches, full `systemd-sysext`/`confext` download/cache/activate/deactivate/remove lifecycle), Advanced OS Installation (`sbm-catalog`), and optional `sbm-server` remote management.
6. **SBM 0.2:** Apple Silicon physical support (`sbm-as`), production Secure Boot / `shim-review`, and public website (`smart-boot-menu.github.io`).

## Consequences

- **Positive:** Unlocks SBM's flagship remote-repair and minimal-EFI use cases without bloating `0.0.1` or inventing a custom package manager or network boot protocol; preserves strict non-destructive storage and revocable security boundaries; unifies desktop provisioning under `SBM Manager`; and turns Track A's native simulator into the daily first-principles UX studio.
- **Trade-offs:** Expanding `SBM Manager` beyond basic USB flashing means the `0.0.3` evaluation must carefully weigh a minimal `rpi-imager` fork against a thin Rust/Iced `SBM Manager` app reusing mature OS flashing backends; maintaining strict `SYSEXT_LEVEL` compatibility and `< 500 MiB` base discipline requires automated size and image-policy gates in CI starting in `0.0.3`.

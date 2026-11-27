# Engineering Targets, Testing Tiers & Benchmark Methodology

**Status:** Normative Baseline (Updated 2026-11-26, aligned with [ADR-011](../architecture-decisions/ADR-011-native-preview-internal-install-and-media-creator.md) & [ADR-012](../architecture-decisions/ADR-012-profiles-network-recovery-modularity-and-ux.md))  
**Important Notice:** All numeric figures in §1 are **initial engineering targets (aspirational design goals)**, NOT measured capabilities. Measured results MUST always be reported separately with exact commit SHAs, hardware/VM configuration, and run counts in `research/validation/evidence/`.

---

## 1. Initial Engineering Targets (Aspirational — Subject to Measurement & Refinement)

| Metric | Initial Engineering Target | Status | Notes |
|---|---|---|---|
| **UEFI initialization (`sbm-uefi` entry to first rendered GOP frame)** | **`< 250 ms`** (stretch: `≤ 100 ms`) | Target (aspirational) | Measured from UEFI entry (`uefi_entry_us`) to first full GOP framebuffer paint on OVMF/HVF and physical x86 hardware (with lazy launch-time UKI verification per Spec 03 §9) |
| **Minimal EFI application size (`efi-only` / `sbm-uefi` stripped PE/COFF)** | **`< 1 MiB`** (stretch: `≤ 750 KiB`) | Target (aspirational) | Compiled `--release`, `lto = true`, `opt-level = "s"` or `"z"`, `panic = "abort"` (with `sha2` crate at `opt-level = 3`) |
| **`sbm-uefi` interactive boot selection to `StartImage`** | **`≤ 50 ms`** | Target (aspirational) | Excluding firmware's own disk I/O and launch-time streaming SHA-256 of recovery UKI |
| **`sbm-uefi` peak heap RAM usage** | **`≤ 32 MiB`** (excl. GOP backbuffer) | Target (aspirational) | Essential for older low-RAM laptops and U-Boot heap limits |
| **`sbm-uefi` remote policy network budget** | **`≤ 3.0 s` default, `≤ 5.0 s` hard cap** | Target (aspirational) | Immediately interruptible (`< 50 ms`) by any local keypress; zero delay when offline / link down |
| **Compressed base management OS (`sbm-core` base, `x86_64` / `aarch64`)** | **`< 500 MiB`** | Target (aspirational) | Minimal immutable EROFS (`zstd`) base + UKI; optional heavy tools split into signed `systemd-sysext` modules starting in SBM 0.0.3 ([ADR-012](../architecture-decisions/ADR-012-profiles-network-recovery-modularity-and-ux.md)) |
| **Compressed `sbm-core` image (`aarch64-apple` Asahi)** | **`≤ 900 MiB`** (hard cap: `≤ 1.5 GiB`) | Target (aspirational) | Super-light Fedora Asahi Minimal `kernel-16k` + modular recovery extensions |
| **Local recovery boot on modern hardware (HVF/KVM / NVMe)** | **`< 10 s`** (stretch: `≤ 6 s`) | Target (aspirational) | Measured from UKI `StartImage` to `labwc` + Iced `sbm-ui` ready |
| **Basic recovery minimum RAM** | **`≤ 2 GiB RAM`** where practical (direct EROFS mode target: `≤ 1.0 GiB`) | Target (aspirational) | RAM-only mode MUST check available memory up front and NEVER spill swap or temp files onto internal SSDs |
| **Native interactive UI preview startup (Track A)** | **`< 1 s`** | Target (aspirational) | Cold/warm launch of native `sbm-gfx`/`sbm-boot` or Iced `sbm-ui` simulator on macOS/Linux host |
| **Warm UI iteration (Track A)** | **`< 5 s`** | Target (aspirational) | Edit UI layout/theme/code → incremental compile → interactive preview visible (with `sccache` + `build-dir`) |
| **Offline independence** | **Zero mandatory internet, account, or cloud service** | Normative Requirement | 100% of local boot, recovery, and offline install features work with NIC disconnected or DNS unavailable |
| **Physical disk safety** | **Zero unauthorized physical disk modifications** | Normative Requirement | Enforced across UEFI, Core, `ram-only` mode, network bootstrap, and all automated test tiers |
| **Build reproducibility** | **Bit-for-bit reproducible UEFI & release media** | Target (aspirational) | Verified via double-build SHA-256 diff (`SOURCE_DATE_EPOCH`, `--remap-path-prefix`, `check-reproducible.sh`) |

> **Current Measured Capabilities (Distinguished from Targets Above):** Measured developer-host build/test benchmarks are archived in `research/validation/evidence/sbm-track-b-benchmarks-2026-11-26.md` and `research/validation/evidence/sbm-dev-resource-audit-2026-11-26.md`. End-to-end SBM 0.0.1 UEFI startup latency, stripped binary size, compressed `sbm-core` size, and graphical cold-boot timings remain **UNQUALIFIED** until reproducible per-architecture 0.0.1 CI artifacts are measured with `N ≥ 10` runs and archived under `research/validation/evidence/`.

---

## 2. Three Mandatory Testing & Qualification Tiers

All SBM verification, CI jobs, and release gates strictly separate three testing tiers so simulated checks are never confused with real disk/firmware mutations, and developer workstations are never exposed to physical disk or NVRAM writes (see [ADR-011](../architecture-decisions/ADR-011-native-preview-internal-install-and-media-creator.md)):

| Tier | Scope & Environment | What It Tests | Safety Boundary | Qualification Authority |
|---|---|---|---|---|
| **Tier 1 — Simulated Installation & Native UI Testing** *(Dev Level 0)* | Native macOS / Linux host; zero VMs, zero root, zero block devices. Uses `cargo test`, `proptest`, regular-file disk images (`sbm-plan`/`sbm-exec`), and Track A interactive native previews (`sbm-gfx`/`sbm-boot` + Iced `sbm-ui` against simulated platform interfaces). | Boot policy, discovery parsing, renderer layout/themes/HiDPI, first-principles UX evaluation, handoff state machines, storage planner safety gates (BitLocker/LUKS/Apple/ESP/free-space/resize math), dry-run diffs, install/uninstall/rollback file logic, and UX flows. | Operates exclusively on in-memory models and temporary regular files. **Visibly badged as simulated.** | Mandatory fast inner-loop, PR gate, and **primary UX evaluation environment**; **NEVER** counts as firmware, kernel, or real block-device qualification. |
| **Tier 2 — Disposable VM Disk Testing** *(Dev Levels 1–4 via `sbm-lab` / `os-autoinst`)* | QEMU + OVMF (`x86_64`), AAVMF (`aarch64`), U-Boot (`qemu_arm64`), and `swtpm` booting **disposable copy-on-write QCOW2 overlays** over immutable fixture disks (multi-OS Windows/Linux ESPs, unallocated space, resizable ext4/NTFS/Btrfs fixtures, BitLocker/LUKS/corrupt GPT fixtures) and fresh disposable EFI NVRAM templates. | Real UEFI boot & chainload, launch-time UKI SHA-256, `dm-verity` EROFS boot, real `labwc` + Iced UI + GParted, real 0.0.2 internal SSD install (`efi-only` & offline Core via `sgdisk`/`sfdisk`/`dosfstools`/`efibootmgr`), real reboot from internal SSD, real uninstall restoring `BootOrder`, real 0.0.3 guided partition resizing & signed `sysext` activation, real 0.1 network recovery / `ram-only` zero-SSD-write assertions, and negative/power-cut fault injection. | Copy-on-write overlays only; original base images and fixtures MUST remain byte-identical after every run. Respects macOS 16 GiB host resource budget (`sbm-lab resources`). | **Primary automated release qualification gate** for SBM 0.0.1, 0.0.2, 0.0.3, and 0.1 (independent per-architecture qualification). |
| **Tier 3 — Explicitly Authorized Physical Hardware Testing** *(Opt-In Lab / `labgrid`)* | Dedicated, explicitly authorized physical test hardware (e.g., older x86 HP laptop `EXP-HW-HP01`, dedicated ARM64 UEFI boards, later M1/M2/M3 test Macs). | Real `SBM Manager` USB flashing, real motherboard UEFI/GOP/USB/Wi-Fi/NVMe quirks, real BitLocker/Windows Update coexistence, real thermal/latency measurements. | **STRICTLY OPT-IN & HUMAN-CONFIRMED.** NEVER automated on developer workstations; NEVER touches the developer's host disks, NVRAM `BootOrder`, or macOS `LocalPolicy`/SIP. | Required before declaring a specific hardware profile *Verified* on public compatibility matrices (`smart-boot-menu.github.io`). |

---

## 3. Benchmark Methodology

1. **Prioritize User-Perceived Latency:** Always report end-to-end wall-clock stages (`firmware_init -> uefi_entry -> discovery_done -> first_frame_painted -> network_check_done -> target_start_image`) rather than synthetic microbenchmarks.
2. **Environment Recording:** Every benchmark artifact saved in `research/validation/evidence/` MUST record: exact git commits of all involved repos, host/guest CPU, QEMU version + accel (`hvf`/`kvm`/`tcg`) or physical hardware profile, firmware version (`edk2-stable202611` / U-Boot), disk backing type, display resolution, and `N ≥ 10` runs (reporting median p50, p95, and max). Single `N = 1` functional integration runs MUST be explicitly labeled as functional runs, not benchmarks.
3. **Zero Silent Regressions:** CI enforces hard ceilings on EFI binary size, compressed `sbm-core` base size (0.0.3+), and boot-smoke timeouts on every PR.

---

## 4. Automated Testing Matrix (Tier 1 & Tier 2)

| Layer | Tier | Tools | Runs On |
|---|---|---|---|
| **Static & Supply Chain** | Tier 1 | `cargo fmt --check`, `cargo clippy --all-targets -D warnings`, `cargo-deny`, `cargo-audit`, JSON Schema validation | Every PR (all repos) |
| **Cross-Compilation** | Tier 1 | `cargo check --target x86_64-unknown-uefi` & `--target aarch64-unknown-uefi` | Every PR (`sbm-common`, `sbm-uefi`) |
| **Unit, Property & Simulated UI/Install Tests** | Tier 1 | `cargo test`, `proptest`, golden test vectors in `sbm-common/test-vectors/`, regular-file `sbm-plan`/`sbm-exec` suites, native interactive preview smoke tests | Every PR |
| **Fuzzing** | Tier 1 | `cargo-fuzz` on BLS parser, PE inspector, `CommandEnvelope` decoder, `StorageOperationPlan` validator | Nightly + pre-release |
| **Virtual Hardware & Disposable Disk Integration** | Tier 2 | QEMU + OVMF (`x86_64`), OVMF Secure Boot + `swtpm`, AAVMF (`aarch64`), U-Boot `qemu_arm64` / U-Boot `sandbox` + QMP / `os-autoinst` visual assertions + disposable multi-OS internal-install/uninstall/resize QCOW2 disks | PR + nightly + release qualification (`sbm-lab` / Linux CI) |
| **Fault Injection** | Tier 2 | QMP power-cut mid-boot/mid-install, TCP blackhole / TLS bad-cert / interrupted HTTP `Range` / replayed command envelope, `dm-flakey` storage errors, tampered UKI / unsigned `sysext` / broken dm-verity / decoy ESP | Nightly + pre-release |
| **Physical Hardware Lab** | Tier 3 | Dedicated opt-in test machines (future `labgrid` coordinator) | Explicit human-authorized runs only |

# ADR-008: Virtualization, Emulation, Fault Injection & Benchmark Methodology

- **Status:** Accepted (SBM 0.1 Baseline)
- **Date:** 2026-11-26
- **Supporting Research:** `research/architecture/virtualization-and-testing.md`

## Context

Pre-boot and disk-management software cannot be safely developed by trial-and-error on physical workstations (especially not on the developer's M5 MacBook Air). We need deterministic, reproducible, multi-architecture virtual testing and honest latency benchmarking.

## Decision

1. **Primary Simulation Harness (`sbm-lab`):**
   - **QEMU** (`qemu-system-x86_64` + `qemu-system-aarch64`) driven by Rust integration test harnesses.
   - **Firmware Matrix:** EDK II (`edk2-stable202611`) **OVMF** (x86-64, standard + Secure Boot), **AAVMF** (ARM64), and **Das U-Boot `v2026.10` (`qemu_arm64`)** (to continuously validate `sbm-uefi` under U-Boot's `CONFIG_EFI_LOADER` subset matching Apple Silicon).
   - **Virtual TPM:** **`swtpm v0.10.2`** attached to QEMU for Measured Boot / PCR / TPM-identity tests.
   - **Golden Disk Fixtures:** Versioned QCOW2 copy-on-write fixtures (empty disk, Windows-layout ESP, multi-Linux BLS/UKI ESP, BitLocker-marked partition, Apple-style protected partition layout, corrupt GPT).
2. **Developer Workstation Workflow (M5 Mac):** Use native-speed `qemu-system-aarch64 -accel hvf` for rapid inner-loop testing and `qemu-system-x86_64 -accel tcg` for x86 smoke tests; run full x86 KVM suites on Linux CI.
3. **Verification Layers:** `cargo test` unit/contract tests -> `sbm-common` golden test vectors -> `proptest` property tests -> `cargo-fuzz` parser fuzzing -> QEMU end-to-end integration + QMP `screendump` visual assertions + power-cut/network/disk fault injection.
4. **Benchmark Discipline:** Prioritize **user-perceived latency** (firmware handoff -> interactive menu paint; menu selection -> target `StartImage`; selection -> `sbm-core` interactive desktop). Never publish performance claims until measured under the documented methodology in `docs/testing/benchmarks.md` and archived under `research/validation/evidence/`.

# Engineering Targets, Testing Matrix & Benchmark Methodology

**Status:** Normative Baseline (2026-11-26)  
**Important Notice:** All numeric figures below are **engineering targets**, NOT measured results. No performance claim may be published until measured using the methodology below and archived in `research/validation/evidence/`.

---

## 1. Initial Engineering Targets

| Metric | Target (SBM 0.1 / v1.0 Goal) | Priority | Notes |
|---|---|---|---|
| **`sbm-uefi` startup to first rendered GOP frame** | **≤ 250 ms** (stretch: ≤ 100 ms) | Critical | Measured from UEFI entry (`uefi_entry_us`) to first full GOP framebuffer paint on OVMF/HVF and physical x86 hardware |
| **`sbm-uefi` interactive boot selection to `StartImage`** | **≤ 50 ms** | Critical | Excluding firmware's own disk I/O for loading the target OS binary |
| **`sbm-uefi` stripped PE/COFF EFI binary size** | **≤ 1.5 MiB** (stretch: ≤ 750 KiB) | High | Compiled `--release`, `lto = true`, `opt-level = "s"` or `"z"`, `panic = "abort"` |
| **`sbm-uefi` peak heap RAM usage** | **≤ 32 MiB** (excl. GOP backbuffer) | High | Essential for older low-RAM laptops and U-Boot heap limits |
| **`sbm-uefi` remote policy network budget** | **≤ 3.0 s default, ≤ 5.0 s hard cap** | Critical | Immediately interruptible (< 50 ms) by any local keypress; zero delay when offline / link down |
| **`sbm-core` compressed image size (`x86_64` / `aarch64`)** | **≤ 650 MiB** (hard cap: ≤ 1.0 GiB) | High | Immutable EROFS (`zstd`) + UKI; well below 4 GiB ceiling |
| **`sbm-core` compressed image size (`aarch64-apple` Asahi)** | **≤ 900 MiB** (hard cap: ≤ 1.5 GiB) | High | Super-light Fedora Asahi Minimal `kernel-16k` + recovery tools; well below 4 GiB ceiling |
| **`sbm-core` cold boot to interactive desktop (HVF/KVM / NVMe)** | **≤ 10 s** (stretch: ≤ 6 s) | High | Measured from UKI `StartImage` to `labwc` + SBM UI ready |
| **`sbm-core` minimum operational RAM (direct EROFS mode)** | **≤ 1.0 GiB** (target 2 GiB older HP laptop) | High | `copytoram` auto-enabled only when RAM ≥ 4 GiB |
| **Offline reliability** | **100% core local features** | Critical | Zero failures or hangs when NIC disconnected, DNS broken, or `sbm-server` unreachable |
| **Build reproducibility** | **Bit-for-bit reproducible `sbm-uefi` EFI** | High | Verified via double-build SHA-256 diff in CI (`SOURCE_DATE_EPOCH`, `--remap-path-prefix`) |

---

## 2. Benchmark Methodology

1. **Prioritize User-Perceived Latency:** Always report end-to-end wall-clock stages (`firmware_init -> uefi_entry -> discovery_done -> first_frame_painted -> network_check_done -> target_start_image`) rather than synthetic microbenchmarks.
2. **Environment Recording:** Every benchmark artifact saved in `research/validation/evidence/` MUST record: exact git commits of all involved repos, host/guest CPU, QEMU version + accel (`hvf`/`kvm`/`tcg`), firmware version (`edk2-stable202611` / U-Boot), disk backing type, display resolution, and `N ≥ 10` runs (reporting median p50, p95, and max).
3. **Zero Silent Regressions:** `sbm-lab` CI enforces hard ceilings on EFI binary size and boot-smoke timeout on every PR.

---

## 3. Automated Testing Matrix (`sbm-lab`)

| Layer | Tools | Runs On |
|---|---|---|
| **Static & Supply Chain** | `cargo fmt --check`, `cargo clippy --all-targets -D warnings`, `cargo-deny`, `cargo-audit`, JSON Schema validation | Every PR (all repos) |
| **Cross-Compilation** | `cargo check --target x86_64-unknown-uefi` & `--target aarch64-unknown-uefi` | Every PR (`sbm-common`, `sbm-uefi`) |
| **Unit & Property Tests** | `cargo test`, `proptest`, golden test vectors in `sbm-common/test-vectors/` | Every PR |
| **Fuzzing** | `cargo-fuzz` on BLS parser, PE inspector, `CommandEnvelope` decoder, `StorageOperationPlan` validator | Nightly + pre-release |
| **Virtual Hardware Integration** | QEMU + OVMF (`x86_64`), OVMF Secure Boot + `swtpm`, AAVMF (`aarch64`), U-Boot `qemu_arm64` + QMP `screendump` | PR + nightly (`sbm-lab`) |
| **Fault Injection** | QMP power-cut mid-boot, TCP blackhole / TLS bad-cert / replayed command envelope, `dm-flakey` storage errors | Nightly + pre-release (`sbm-lab`) |

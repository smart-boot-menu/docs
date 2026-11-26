# ADR-004: `sbm-uefi` Rust Stack, Opportunistic Networking & ESP-First Persistence

- **Status:** Accepted (SBM 0.1 Baseline)
- **Date:** 2026-11-26
- **Supporting Research:** `research/architecture/uefi-rust-stack.md`

## Context

`sbm-uefi` must compile cleanly from a single Rust codebase for `x86_64-unknown-uefi` and `aarch64-unknown-uefi` (with best-effort `i686-unknown-uefi` compatibility), handle buggy consumer UEFI firmwares safely, support optional remote boot policy checks, and never brick user motherboards via NVRAM corruption.

## Decision

1. **Language & Core Crate:** **Rust 2024 edition** (`#![no_std]` + `alloc`) using **`uefi` v0.41.0+** (`uefi-rs` global module API `uefi::boot`, `uefi::system`, `uefi::runtime`).
2. **Testability Architecture:** Isolate all UEFI calls behind thin Rust traits (following Google `crdyboot`'s pattern) so OS discovery, BLS parsing, policy resolution, and UI layout run natively under `cargo test` on the host.
3. **OS Discovery Standards:** Implement UAPI **Boot Loader Specification (BLS) Type #1 (`.conf`) and Type #2 (UKI)** alongside heuristic EFI binary scanning (Windows Boot Manager, Linux distros, BSD loaders, `sbm-core` A/B slots) on standard FAT32 ESP/XBOOTLDR partitions without requiring third-party EFI filesystem drivers.
4. **Defensive Input Handling:** Use a bounded periodic timer tick (16–33 ms) + non-blocking polling of `SimpleTextInputEx`, `SimplePointer`, and `AbsolutePointer` to avoid `WaitForKeyEx` firmware hangs; guarantee 100% keyboard usability.
5. **Opportunistic Bounded Networking:** Use Rust **`smoltcp` + TLS 1.3 (`embedded-tls` / `rustls` `no_std`) over UEFI SNP** (with optional native `EFI_HTTP_PROTOCOL` adapter), strictly bounded to **≤ 4.0 s total timeout**, immediately cancellable by any keypress, and strictly limited to read-only policy fetch + boot-attempt reporting (**zero arbitrary RCE in UEFI**).
6. **ESP-First Persistence (No Frequent NVRAM Writes):** Persist all SBM configuration, policy caches, and boot-attempt counters atomically on the ESP (`\EFI\sbm\...`). Touch UEFI NVRAM only for installer `BootOrder` registration and volatile read-only handoff variables (`LoaderEntrySelected`, SBM vendor GUID).
7. **Secure Boot Readiness:** Embed a valid `.sbat` PE section and `IMAGE_DLLCHARACTERISTICS_NX_COMPAT` from early milestones to stay compliant with `rhboot/shim 16.1+` and Microsoft UEFI CA 2023 signing rules.

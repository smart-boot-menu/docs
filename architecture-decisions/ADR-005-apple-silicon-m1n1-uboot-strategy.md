# ADR-005: Apple Silicon Integration (`m1n1` + Asahi U-Boot + Shared `sbm-uefi`)

- **Status:** Accepted (Architecture) / M5 Hardware Deferred Pending Upstream Drivers
- **Date:** 2026-11-26
- **Supporting Research:** `research/hardware/apple-silicon-m1-to-m5.md`

## Context

Apple Silicon uses a closed Boot ROM -> LLB -> `LocalPolicy` -> iBoot chain rather than native UEFI. Furthermore, as of November 2026, Asahi Linux officially supports M1, M2, and M3, while M4 is in early bring-up and **M5 (`T8142`) lacks USB, Thunderbolt, PCIe, DART, GPU, Wi-Fi, and Touchpad drivers (`TBA`)**.

## Decision

1. **Reuse `m1n1` + `AsahiLinux/u-boot` (`asahi-releng`):** `sbm-as` integrates upstream `m1n1` (blessed as an iBoot custom boot object via 1TR `bputil`) chainloading `AsahiLinux/u-boot`, which exposes UEFI Boot Services (`CONFIG_EFI_LOADER`) and boots the **exact same `aarch64-unknown-uefi` `sbm-uefi` binary** used on standard ARM64 PCs.
2. **No `sbm-uefi` Fork:** Platform differences (e.g., lack of writable UEFI NVRAM, `VIDEO_SIMPLE` framebuffer mode preservation, no direct EFI chainloading of macOS) are handled cleanly via `PlatformCapabilities` in `sbm-common` and platform adapters in `sbm-as`.
3. **macOS Switching Semantics:** On Apple Silicon, selecting macOS from SBM sets the Apple boot target / triggers a reboot into Apple's startup chain rather than attempting impossible EFI chainloading of macOS.
4. **Internal Stub -> External Storage Boot Design:** `sbm-as` targets a compact internal stub (~500 MB APFS stub + 512 MB ESP + optional < 1 GB `sbm-core-aarch64-apple`, total ~1.5–2 GB internal) that can boot compatible ARM64 OSes (`linux-asahi` distros) from external **USB 3.x (xHCI/UASP)** SSDs on supported generations (M1/M2/M3 today; M4/M5 once upstream Asahi lands DART/PCIe/USB3).
5. **Absolute Hardware Safety:** `sbm-as` and `sbm-core` MUST permanently classify Apple `iSCPreboot`, `RecoveryOS`, and macOS APFS containers as **platform-critical protected partitions** (read-only / blocked from modification). Never modify the development M5 MacBook Air's boot policy or partitions.
6. **SBM 0.1 Scope:** Validate the U-Boot UEFI execution path using **QEMU `qemu_arm64` U-Boot** inside `sbm-lab`; defer physical M5 bare-metal bring-up until upstream `linux-asahi` / `m1n1` / `u-boot` supports M5 (`T8142`) USB/PCIe/DART.

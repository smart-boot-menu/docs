# ADR-003: `sbm-core` Minimal Linux OS — Unified `mkosi` (Fedora Minimal for x86/ARM64 PCs + Light Fedora Asahi Minimal for Apple Silicon)

- **Status:** Accepted (SBM 0.1 Baseline)
- **Date:** 2026-11-26
- **Supporting Research:** `research/architecture/minimal-linux-core-os.md`, `research/hardware/apple-silicon-m1-to-m5.md`

## Context

`sbm-core` must run reliably across:
1. **x86-64 PCs** (from older HP laptops with 2–4 GB RAM to modern workstations with Wi-Fi 7 & NVMe),
2. **Generic ARM64 UEFI systems** (QEMU AAVMF, ARM laptops/servers), and
3. **Apple Silicon Macs** (which require Asahi Linux's `linux-asahi` 16 KiB-page kernel, devicetrees, and Mesa/firmware ecosystem, while staying compact — well below a 4 GB ceiling).

It must be **immutable, modular, independently recoverable, RAM-resident capable, and capable of running GParted, Wi-Fi, OS installers, and remote desktop**.

## Alternatives Considered

1. **Buildroot (`2026.08.2`) / Yocto:** Source-compiling Mesa, GTK3/gtkmm3, GParted, Wayland, XWayland, and `linux-asahi` creates 45–90+ minute builds and severe maintenance overhead. Rejected.
2. **Alpine Linux (musl + BusyBox + OpenRC):** Very compact (~250 MB), but lacks native `systemd-sysext`/`systemd-sysupdate`/`systemd-stub` UKI ecosystem, encounters musl edge cases with GTK3/Mesa/third-party binaries, and has no first-class official Asahi Linux maintenance. Kept only as a future research note if ultra-low-RAM (< 512 MB) hardware becomes a priority.
3. **Debian Minimal (`mmdebstrap`/`mkosi`) for x86 + separate Fedora Asahi for Apple Silicon:** Forces maintaining two different distro package lists, kernel packaging models, and systemd versions.
4. **Unified `mkosi v27.1` Pipeline using Fedora Minimal (x86-64 & generic ARM64) + Fedora Asahi Minimal (Apple Silicon) (Selected):**
   - **x86-64 & generic ARM64 PCs:** `mkosi` builds a stripped **Fedora Minimal** immutable **EROFS (`zstd`)** rootfs + **Unified Kernel Image (UKI)** (~420–650 MB compressed). Uses fresh upstream mainline kernels for broad PC/Wi-Fi/GPU compatibility.
   - **Apple Silicon:** The **exact same `mkosi` profile**, switching only the platform package layer to **Fedora Asahi Remix repositories** (`kernel-16k` `linux-asahi`, `mesa-asahi`, `asahi-scripts`, `m1n1`, `uboot-asahi`) — producing a **super-light Asahi `sbm-core` of ~500–850 MB compressed** (well under **1 GB**, < 25% of the 4 GB budget!).

## Decision

Standardize `sbm-core` on **`mkosi v27.1`** producing **signed Unified Kernel Images (UKIs) + `dm-verity`-verified EROFS (`zstd`) root filesystems**, with:
- **Fedora Minimal** for `x86_64` and generic `aarch64` UEFI PCs (target compressed size: **400–650 MB**),
- **Fedora Asahi Minimal (`kernel-16k`)** for Apple Silicon (target compressed size: **500–850 MB**, never redistributing Apple firmware blobs — extracted locally via `asahi-fwextract`),
- Curated `.zst`-compressed `linux-firmware` core subset + optional **`systemd-sysext`** modular extensions (`gparted`, `diagnostics`, `firmware-extra`, `zfs`, `remote-desktop`),
- Automatic **RAM-copy (`sbm.toram=auto`)** when RAM ≥ 4 GiB, and direct EROFS + `zram` overlay fallback when RAM < 4 GiB,
- **A/B atomic slots** managed via `systemd-sysupdate` + `sbm-uefi` automatic boot-attempt rollback.

## Open Experiment

- **EXP-CORE-01:** Build the SBM 0.1 `x86_64` and `aarch64` `mkosi` images in `sbm-core`, verify exact compressed size (< 650 MB), boot time in QEMU (< 8 s on HVF/KVM), and GParted launch under `labwc` + XWayland.

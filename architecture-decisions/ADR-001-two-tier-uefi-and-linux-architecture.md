# ADR-001: Two-Tier Execution Architecture (UEFI Fast Path + Linux Management Path)

- **Status:** Accepted (Foundational)
- **Date:** 2026-11-26
- **Supporting Research:** `research/ecosystem/boot-management.md`, `research/architecture/uefi-rust-stack.md`

## Context

Smart Boot Menu must provide both:
1. Sub-second graphical boot selection and OS chainloading (Windows, Linux, BSD, EFI utilities) every time a computer boots.
2. Rich hardware-dependent management features: Wi-Fi/WPA3 networking, GPU/Wayland graphics, GParted, partition resizing, OS installation, backup/cloning, remote desktop, and diagnostics.

## Alternatives Considered

1. **Linux-only (`kexec`) boot manager (Petitboot / ZFSBootMenu model):**
   - *Pros:* Full Linux driver stack immediately available.
   - *Cons:* Cannot chainload Windows (`bootmgfw.efi`) or arbitrary EFI applications after `ExitBootServices()` has been called; adds 3–10+ seconds of kernel init to every normal boot; high RAM/storage overhead for routine booting.
2. **UEFI-only monolithic boot manager (rEFInd / GRUB / BootIt UEFI model):**
   - *Pros:* Fast startup, native EFI chainloading.
   - *Cons:* UEFI lacks modern Wi-Fi drivers, GPU acceleration, mature filesystem resizers (ext4/NTFS/Btrfs/LUKS2/BitLocker), GParted, and full remote desktop stacks; writing complex disk/crypto code in UEFI C/Rust is brittle and unsafe.
3. **Two-Tier Architecture (Selected):**
   - **Tier 1 — `sbm-uefi` (Fast Path):** Minimal native Rust UEFI application (`x86_64-unknown-uefi`, `aarch64-unknown-uefi`) handling graphical boot menu, OS discovery, EFI chainloading, local/remote boot policies, opportunistic wired networking, and launching Tier 2.
   - **Tier 2 — `sbm-core` (Management Path):** Independently bootable, immutable, RAM-capable minimal Linux environment launched as a Unified Kernel Image (UKI) only when the user or remote policy requests management, installation, recovery, or diagnostics.

## Decision

Adopt the **two-tier architecture (`sbm-uefi` + `sbm-core`)**. Normal OS booting NEVER initializes Linux. `sbm-uefi` and `sbm-core` communicate via a strict, versioned EFI-to-Linux handoff contract (`docs/specifications/03-efi-linux-handoff-and-recovery.md`).

## Consequences

- **Positive:** Sub-second everyday boots; 100% compatibility with Windows/Linux/BSD EFI chainloading; full Linux driver & GParted/Partclone/wimlib power when managing the system; clean security boundary (no arbitrary RCE or destructive disk writes in UEFI).
- **Trade-offs:** Must maintain two runtimes (UEFI `no_std` Rust + Linux `std` Rust) and a well-tested handoff protocol between them.

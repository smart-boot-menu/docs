# Target Hardware Profiles & Experimental Roadmap

**Last verified:** 2026-11-26  
**Supporting Research:** `research/hardware/apple-silicon-m1-to-m5.md`, `research/architecture/virtualization-and-testing.md`

---

## Target 1: Older x86 HP Laptop (Physical Target — Pending Hardware Probe)

- **Current Status:** Exact model, CPU, RAM, UEFI version, Secure Boot key store (2011 vs 2023 CAs), Wi-Fi chipset, and GOP/input quirks have **not yet been probed**.
- **Expected Platform Profile:**
  - `arch`: `x86_64` (verify whether 64-bit UEFI or quirky 32-bit IA32 UEFI!)
  - `firmware`: `uefi-native` (likely InsydeH2O or AMI Aptio — high probability of `WaitForKeyEx` / NVRAM / I2C trackpad quirks)
  - `sbm-core` profile: `sbm-core-x86_64` (Fedora Minimal via `mkosi`, EROFS + `zstd`, ~420–650 MB). If RAM < 4 GiB, mounts EROFS directly with `zram` overlay instead of `copytoram`.
- **Hardware Characterization Experiment (`EXP-HW-HP01`):**
  1. Boot an unmodified read-only live Linux USB (or SBM 0.1 removable USB) without touching internal storage.
  2. Collect (redacting serial numbers/MACs): `dmidecode` model/BIOS version, `lscpu`, `free -m`, `lspci -nnk` (GPU, Wi-Fi, NVMe/SATA), `lsusb`, `efibootmgr -v`, `mokutil --sb-state`, `efi-readvar -v db` (check 2011 vs 2023 Microsoft CA presence), GOP modes, and whether trackpad works in UEFI `SimplePointer`.
  3. Record redacted hardware profile in `research/hardware/hp-laptop-profile.md`.

---

## Target 2: Apple Silicon M5 MacBook Air (`T8142` — Development Workstation)

- **Observed Hardware (2026-11-26, Read-Only Inspection):**
  - CPU / SoC: **Apple M5** (`ARM64_T8142`)
  - OS: **macOS 27.0** (Darwin `27.0.0`, build `26A428`)
  - Toolchain verified: `rustc 1.100.0`, `cargo 1.100.0`, targets `aarch64-apple-darwin`, `wasm32-unknown-unknown`, `x86_64-unknown-uefi`, `aarch64-unknown-uefi`, `gh 2.101.0`, `brew`, `python3`, `node`, `jq`.
  - Virtualization tools to install when Agent 3 (`sbm-lab`) begins: `brew install qemu swtpm`.
- **ABSOLUTE SAFETY RULE:** **NEVER modify this development workstation's macOS boot configuration, LocalPolicy, SIP, NVRAM boot-args, or physical NVMe partitions.**
- **Verified Bare-Metal Linux / Asahi Support Status (Nov 2026):**
  - **Verified Working (`linux-asahi` 7.5 target):** Devicetree, boot framebuffer (`simpledrm`), internal NVMe, keyboard & backlight, AICv3, UART, Watchdog, I²C, GPIO.
  - **NOT Supported Yet (`TBA`):** Official Asahi installer (`no`), USB2, USB3, Thunderbolt, PCIe, DART, GPU, DCP, Wi-Fi, Bluetooth, Touchpad, Battery, Brightness, Speakers/Mic, Webcam, Suspend.
  - **Firmware Caveat (`FB23916610`):** On M5 Pro/Max/Ultra, macOS 27.3 Golden Gate Beta (`mBoot-20457.60.65.0.1`) fixed a single-core custom raw boot object bug; base M5 `T8142` still awaits full peripheral drivers in `linux-asahi`.
- **Consequence for SBM 0.1:**
  - Use the M5 MacBook Air as a **blazing-fast native ARM64 virtualization host** (`qemu-system-aarch64 -accel hvf` with AAVMF and `qemu_arm64` U-Boot) and cross-compiler for `x86_64-unknown-uefi` + `aarch64-unknown-uefi`.
  - Physical Apple Silicon testing should start on a **supported M1/M2/M3 test Mac** (where `m1n1`, Asahi U-Boot, USB3 external boot, and `linux-asahi` are verified stable) or wait until upstream Asahi lands M5 (`T8142`) USB3/DART/PCIe support.

---

## Open Apple Silicon External-Boot Experiment (`EXP-AS-EXT01`)

- **Hypothesis:** A compact internal Apple Silicon stub (~1.5–2 GB total: Apple APFS stub + 512 MB ESP + `m1n1` + `AsahiLinux/u-boot` + `sbm-uefi.efi` + optional `< 1 GB` `sbm-core-aarch64-apple`) can discover and boot a `linux-asahi` OS installed entirely on an external **USB 3.2 Gen 2 (UASP/xHCI)** SSD without consuming further internal NVMe space.
- **Validation Path:**
  1. Phase 1 (Safe VM validation in `sbm-lab`): Boot `sbm-uefi` (`aarch64-unknown-uefi`) under QEMU **`qemu_arm64` U-Boot** with an internal virtio-blk ESP and a hot-attached virtual USB xHCI SSD containing a BLS/UKI Linux install.
  2. Phase 2 (Physical M1/M2/M3 test hardware — NEVER the dev M5 Mac until M5 USB3/DART lands): Validate U-Boot xHCI enumeration of external USB-C SSDs and boot-to-desktop latency.

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
  - `qemu-system-aarch64` 11.1.3 is installed. On 2026-11-26 its HVF accelerator aborted (`hvf_handle_exception`) before the probe ran. The passing U-Boot EFI probe used TCG (`-machine virt -cpu cortex-a57`). HVF is not the proven path.
- **ABSOLUTE SAFETY RULE:** **NEVER modify this development workstation's macOS boot configuration, LocalPolicy, SIP, NVRAM boot-args, or physical NVMe partitions.**
- **Asahi feature table for `T8142` (fetched 2026-11-26):**
  - Installer `no`. This Mac is not an install target.
  - Marked `linux-asahi` or `linux-asahi (7.5)`: devicetree, boot framebuffer, internal NVMe, keyboard, keyboard backlight, AICv3, UART, watchdog, I²C, GPIO. `(7.5)` is the upstream target on that page. The newest `AsahiLinux/linux` tag the same day is `v7.3.4`.
  - `TBA`: USB2, USB3, Thunderbolt, PCIe, DART, GPU, DCP, Wi-Fi, Bluetooth, touchpad, battery, brightness, audio, webcam. PMU is `WIP`.
  - M5 Pro/Max/Ultra (`T6050`, and `T6051` where the device list uses it) are a separate, still-`TBA` column. Apple feedback `FB23916610` was not fetched from Apple and is not a `T8142` result.
- **Consequence for SBM 0.1:**
  - Use the Mac as a build machine and as the TCG host for the QEMU `virt` U-Boot probe. Do not change its boot policy or partitions.
  - Physical Apple Silicon testing starts on a machine whose installer cell is `yes` (M1, M2 except Mac Pro 2023, M3 except Ultra). M5 waits on upstream USB, DART, and PCIe.

---

## Open Apple Silicon External-Boot Experiment (`EXP-AS-EXT01`)

- **Hypothesis:** On a machine the Asahi installer already supports, an internal m1n1 plus U-Boot bootstrap can start the shared aarch64 SBM UEFI binary, which can then see a USB disk after U-Boot runs `usb start`. The 1.5–2 GB internal footprint is a target, not a measurement. A generic external image is not enough; the OS still needs the Apple device tree and a 16 KiB page kernel.
- **Validation Path:**
  1. Phase 1, done 2026-11-26 under QEMU `virt` (not Apple hardware, and not xHCI): `sbm-as/scripts/uboot-efi-poc.sh` started the probe, filled a GOP rectangle, wrote the ESP policy file, chainloaded a second EFI application, and found a file on a second USB-EHCI disk. Evidence: `research/validation/evidence/sbm-as-uboot-efi-2026-11-26.md`.
  2. Phase 2 stays on physical M1/M2/M3 test hardware. It does not run on this M5 Mac.

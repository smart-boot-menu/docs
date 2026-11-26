# Spec 01: Identifiers & Platform Capabilities (`v1`)

**Status:** Normative Draft v1.0 (`sbm-types`, `#![no_std]` + `alloc`)

## 1. CPU Architecture Identifiers (`CpuArch`)

Canonical lowercase string identifiers (used in manifests, catalog entries, policies, and capability reports):

| Variant | Wire String | Notes |
|---|---|---|
| `X86_64` | `"x86_64"` | 64-bit x86 (AMD64) |
| `Aarch64` | `"aarch64"` | 64-bit ARM (ARMv8/v9, including Apple Silicon) |
| `Ia32` | `"ia32"` | 32-bit x86 UEFI where practical |
| `Riscv64` | `"riscv64"` | Future expansion |

## 2. Firmware Platform Identifiers (`FirmwarePlatform`)

| Variant | Wire String | Description |
|---|---|---|
| `UefiNative` | `"uefi-native"` | Standard PC/server/laptop UEFI firmware (EDK2/AMI/Insyde/Phoenix) |
| `UefiUBoot` | `"uefi-uboot"` | Generic Das U-Boot `CONFIG_EFI_LOADER` UEFI layer |
| `AppleSiliconM1n1UBoot` | `"apple-silicon-m1n1-uboot"` | Apple iBoot -> `m1n1` -> Asahi U-Boot UEFI chain |
| `LegacyBios` | `"legacy-bios"` | Reserved for future legacy BIOS support |

## 3. Platform Capabilities (`PlatformCapabilities`)

Reported by `sbm-uefi`, `sbm-core`, and `sbm-agent` so UI, policies, and `sbm-server` never attempt unsupported actions:

- `arch: CpuArch`
- `firmware: FirmwarePlatform`
- `page_size_bytes: u32` (e.g., `4096` on x86-64/generic ARM64, `16384` on Apple Silicon)
- `secure_boot_state: SecureBootState` (`Disabled | Enabled | SetupMode | Unsupported`)
- `tpm_present: bool`
- `nvram_writable: bool` (`false` on Apple Silicon U-Boot and quirk-flagged firmwares)
- `gop_framebuffer: bool`
- `uefi_networking: bool`
- `can_chainload_efi: bool`
- `can_direct_chainload_macos: bool` (`false` on Apple Silicon — requires Apple boot-target reboot handoff)
- `external_usb_boot_supported: bool` (`false` on Apple M5 until upstream DART/PCIe/USB3 lands)
- `ram_total_mib: Option<u64>`

## 4. Boot Targets & Persistent Identifiers (`BootTarget`, `BootTargetId`)

A `BootTargetId` MUST be **stable across reboots and disk enumeration order changes**. Never derive persistent IDs from `/dev/sdX` or ephemeral UEFI handle pointers.

- **Canonical Format:** Lowercase deterministic slug or UUID-backed string:
  - Discovered EFI loader: `efi:<esp_part_gpt_guid>:<sha256_prefix_12_of_normalized_efi_path>`
  - BLS Type #1 / Type #2 (UKI): `bls:<esp_part_gpt_guid>:<entry_id>`
  - SBM Core Recovery Slot: `sbm-core:slot-a` / `sbm-core:slot-b` / `sbm-core:removable`
  - Apple macOS volume (reboot target): `apple-macos:<apfs_volume_group_uuid>`
- **Fields of `BootTarget`:**
  - `id: BootTargetId`
  - `display_name: String`
  - `os_family: OsFamily` (`Linux | Windows | MacOs | Bsd | Recovery | FirmwareUtility | Other`)
  - `kind: BootTargetKind` (`EfiBinary | BlsType1 | BlsUki | SbmCore | AppleMacOsReboot | FirmwareSetup`)
  - `arch: CpuArch`
  - `partition_guid: Option<String>` (GPT Partition GUID)
  - `efi_path: Option<String>` (normalized backslash or forward-slash ESP-relative path, e.g. `\EFI\Microsoft\Boot\bootmgfw.efi`)
  - `kernel_cmdline: Option<String>`
  - `icon_hint: Option<String>`
  - `removable_media: bool`
  - `hidden: bool`
  - `verified_signature: Option<bool>`

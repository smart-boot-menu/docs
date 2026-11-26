# Apple Silicon boot architecture

**Status:** Requirement for `sbm-as`. Accepted direction remains ADR-005, as amended 2026-11-26.

**Evidence:** `research/hardware/apple-silicon-m1-to-m5.md`, `research/hardware/apple-silicon-external-boot.md`.

SBM does not bypass Apple boot security. The development M5 Mac is a build host. Nothing in this design writes its internal storage, LocalPolicy, SIP, or NVRAM.

## Sequence

On a machine the Asahi installer supports:

1. Apple Boot ROM, LLB, and iBoot load the SEP-backed LocalPolicy for the chosen OS and the m1n1 stage-1 object installed from paired one-true recovery.
2. Stage 1 loads `/m1n1/boot.bin` from the internal System ESP. That file is m1n1 stage 2, the device tree, and U-Boot.
3. U-Boot starts `\EFI\BOOT\BOOTAA64.EFI`. That binary is the shared aarch64 SBM UEFI application when `sbm-uefi` builds it. Until then, `sbm-as-efi-probe` is the protocol stand-in, not a boot manager.
4. SBM `StartImage`s another aarch64 EFI application, or records an Apple volume as a reboot target and returns to the Apple picker.

Stage 2 does not come from USB. USB disks become visible to the EFI application only after U-Boot has started the USB stack. The pinned Asahi defconfig does not do that before `bootflow scan -b`. `sbm-as/config/apple-uboot.fragment` sets `CONFIG_PREBOOT="usb start"` for a future hardware build. It is not applied to any Mac by the proof script.

## Platform adapter

`sbm-as-platform` projects the existing `PlatformCapabilities` fields. No schema change.

| Field | Apple bridge |
|---|---|
| `firmware` | `AppleSiliconM1n1Uboot` |
| `arch` | `Aarch64` |
| `page_size_bytes` | 16384 |
| `secure_boot_state` | `Unsupported` (UEFI Secure Boot is not the trust root; LocalPolicy is) |
| `tpm_present` | false |
| `nvram_writable` | false |
| `uefi_networking` | false (`CONFIG_NO_NET`) |
| `can_direct_chainload_macos` | false |
| `can_chainload_efi` | true only as "U-Boot is up" |
| `gop_framebuffer` | true when that SoC's boot-framebuffer cell is published |
| `external_usb_boot_supported` | installer `yes`, USB3 published, DART not `WIP`/`TBA` |

`bridge_reachable` is separate: installer `yes` and NVMe published. Installation UI must check it. M4 and M5, including `T8142` and `T6050`/`T6051`, are not reachable.

Boot decisions:

- macOS and `AppleMacOsReboot` on this firmware become a reboot to Apple. The volume-group id is a lowercase UUID. SBM does not start `boot.efi`.
- An aarch64 `EfiBinary`, BLS, UKI, or `sbm-core` target may be chainloaded when `can_chainload_efi` is true.
- Windows, an architecture mismatch, and a UEFI setup application are rejected.
- The Linux handoff needs the Apple device tree U-Boot already passed. APFS is not an EFI filesystem.

## Graphics, input, storage, variables

GOP comes from U-Boot's simple framebuffer (`CONFIG_VIDEO_SIMPLE`, `CONFIG_NO_FB_CLEAR`). The menu must not switch GOP mode. The QEMU proof uses Bochs VGA to publish a GOP of the same protocol; that driver is not the Apple one.

The pinned defconfig has internal Apple keyboards and USB keyboards. It has no USB mouse or trackpad driver, so a pointer protocol is not part of the Apple contract.

Block IO and `SimpleFileSystem` cover the ESPs U-Boot has enumerated. The policy file is `\EFI\sbm\boot-policy.json`. That file is the persistence path already required by the EFI handoff specification. `BootOrder` cannot select an OS.

The device tree is forwarded. The System ESP is identified by `asahi,efi-system-partition`, not by "the first ESP." One OS owns `/m1n1/boot.bin`.

## Partitions the installer must leave alone

`sbm-as-platform` classifies these as `ProtectedAppleSystem`, which `StorageOperationPlan::blocks_by_default_safety` already blocks:

- GPT type Apple APFS ISC, Apple APFS, and Apple APFS Recovery
- content names `Apple_APFS_ISC`, `Apple_APFS`, `Apple_APFS_Recovery`
- container names `iBootSystemContainer` and `RecoveryOSContainer`
- the `iSCPreboot` volume
- APFS roles System, Data, Preboot, Recovery, VM, Update, xART, Hardware, and Prelogin

A System ESP, or an ESP that already hosts a bootloader, is `ProtectedEsp`. A blank ESP is `Normal`. The whole-container Apple APFS volumes, including an external macOS container and the Asahi stub, stay protected. Uninstall is its own flow. It is not a generic delete of those partitions.

## QEMU is not the Mac

`scripts/uboot-efi-poc.sh` builds upstream U-Boot `v2026.10` (`a58e2b7af42c179ace2c6210de0a219fa3f3cf3c`) with `qemu_arm64_defconfig` plus `config/uboot-qemu-efi.fragment`, and boots the probe on `qemu-system-aarch64 -machine virt`. That machine has virtio storage and USB EHCI. It does not have Apple NVMe, Apple DART, xHCI, or the Apple boot ROM. A pass there shows the EFI protocols. It does not show M1–M5 hardware support.

## Recovery image

The Apple `sbm-core` image, when the shared mkosi pipeline builds it, uses a 16 KiB page kernel, the device tree from the boot chain, and firmware extracted on the target Mac with `asahi-fwextract`. The image does not redistribute Apple firmware. Graphics are the Asahi Mesa driver only where the GPU cell is `linux-asahi` (M1 and M2). Elsewhere the image uses the boot framebuffer or a software renderer. There is no UEFI network device. The profile notes live in `sbm-core`; this repository does not build a second distribution.

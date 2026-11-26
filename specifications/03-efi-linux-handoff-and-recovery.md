# Spec 03: EFI-to-Linux Handoff & Recovery Discovery (`v1`)

**Status:** Normative Draft v1.0 (`sbm-types` + `sbm-protocol`, `#![no_std]` + `alloc`)

## 8. EFI-to-Linux Handoff (`EfiHandoffPayload`)

When `sbm-uefi` launches `sbm-core` (or a Linux target with `sbm-agent`), context is passed cleanly across `ExitBootServices()` using **two complementary mechanisms** (so it works identically on EDK2 UEFI AND Apple Silicon U-Boot where NVRAM is non-persistent):

1. **Primary (Universal, ESP-backed):** Atomic JSON file at `\EFI\sbm\handoff\current.json` (plus SHA-256 hash passed via kernel cmdline parameter `sbm.handoff_sha256=<hex>` and `sbm.esp_partuuid=<gpt-guid>`).
2. **Secondary (UAPI Boot Loader Interface volatile EFI variables under vendor GUID `d8f4a21c-7b6e-4c91-9e32-8a5c1f0e3b40` + standard systemd GUID `4a67b082-0a4c-41cf-b6c7-440b29bb8c4f`):**
   - `LoaderEntrySelected`, `LoaderTimeInitUSec`, `LoaderTimeExecUSec`
   - `SbmHandoffVersion` (UTF-16 `"1.0"`)
   - `SbmBootAttemptId` (UTF-16 attempt ID)

### `EfiHandoffPayload` Fields
- `schema_version: "1.0"`
- `attempt_id: String`
- `sbm_uefi_version: String`
- `platform: PlatformCapabilities`
- `launch_mode: RecoveryLaunchMode` (`Interactive | RemoteAssist | DiagnosticOnly | PlannedOperation`)
- `pending_command_id: Option<String>`
- `discovered_targets: Vec<BootTarget>` (snapshot of what `sbm-uefi` saw pre-boot)
- `boot_timing_us: Option<BootTimingMetrics>` (`firmware_init_us`, `uefi_entry_us`, `first_frame_us`, `network_us`, `handoff_us`)
- `network_hint: Option<NetworkHandoffHint>` (non-secret DHCP/link status discovered by UEFI; NEVER plaintext Wi-Fi passwords!)

## 9. Recovery Environment Discovery (`RecoverySlotDescriptor`)

`sbm-uefi` discovers `sbm-core` instances deterministically without scanning arbitrary untrusted filesystems:
1. Inspects `\EFI\sbm\recovery\manifest.json` on the active ESP (and any removable SBM media ESP).
2. Verifies each slot (`SlotA`, `SlotB`, `Removable`) references a **Unified Kernel Image (UKI)** path (e.g., `\EFI\sbm\recovery\slot-a\sbm-core-x86_64.efi`) and expected SHA-256 digest.
3. Under Secure Boot, delegates PE/COFF signature verification to UEFI firmware / `shim` Loader Protocol v2 before `StartImage`.
4. **Independent Recoverability Guarantee:** If `SlotA` fails health confirmation (`max_failed_attempts` reached) or fails hash/signature check, `sbm-uefi` automatically selects `SlotB` (or prompts for removable SBM recovery media).

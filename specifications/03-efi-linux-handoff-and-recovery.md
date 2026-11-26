# Spec 03 — EFI-to-Linux handoff and recovery discovery

**Status** Normative draft for protocol 1.0. The 0.0.1 profile covers removable media on x86_64 OVMF and AArch64 AAVMF.

## 8. EFI-to-Linux handoff

UEFI writes the compact UTF-8 JSON representation of the shared EfiHandoffPayload to \EFI\sbm\handoff\current.json on its boot ESP. The payload schema stays at 1.0. Use interactive launch mode with no pending command. The payload carries the attempt ID, UEFI version, platform, discovered-target snapshot, and optional timing and network fields. Runtime data must exclude secrets.

Before starting the UKI, UEFI must persist the attempt per Spec 02, flush the handoff file, and publish the following volatile variables with boot-service and runtime access. Text values are NUL-terminated UTF-16.

| Variable | Vendor GUID | Value |
|---|---|---|
| LoaderDevicePartUUID | 4a67b082-0a4c-41cf-b6c7-440b29bb8c4f | GPT partition UUID of the boot ESP |
| LoaderEntrySelected | 4a67b082-0a4c-41cf-b6c7-440b29bb8c4f | sbm-core:slot-a |
| SbmHandoffVersion | d8f4a21c-7b6e-4c91-9e32-8a5c1f0e3b40 | 1.0 |
| SbmBootAttemptId | d8f4a21c-7b6e-4c91-9e32-8a5c1f0e3b40 | Attempt ID in current.json |
| SbmHandoffSha256 | d8f4a21c-7b6e-4c91-9e32-8a5c1f0e3b40 | Lowercase hex SHA-256 of the exact current.json bytes |

On write or publication failure, report the error and return to the picker without starting Core. On Linux, resolve the ESP by the BLI PARTUUID, verify VFAT and GPT ESP type and that its parent is the boot medium supplying the verity pair, and revalidate an existing mount before reading or writing through it. In RAM mode carry the resolved boot-media identity across switch-root. Resolve ambiguity, missing identity, or a mismatched parent as an error without selecting the first ESP. Require the schema version, attempt ID, native platform, and digest to agree. Missing or invalid handoff yields a local diagnostic and no success report. The digest checks consistency across boot stages and does not independently authenticate the ESP.

Standalone development images may still display legacy handoffs delivered through sbm.handoff_sha256 and sbm.esp_partuuid. Standalone results do not satisfy the integrated handoff or confirm an integrated attempt.

## 9. Recovery discovery and bootable USB media

### Manifest

Read \EFI\sbm\recovery\manifest.json on the boot ESP. Use the shared RecoveryManifest object with schema_version, active_slot, and slots. Each RecoverySlotDescriptor has slot_name, uki_efi_path, sha256, version, and healthy.

The 0.0.1 schema version is 1.0, the active slot is slot-a, and the producer writes one slot-a descriptor per architecture, using the canonical paths below. A descriptor for the other architecture has the same slot name. Validate the exact path and PE machine for the native entry. A duplicate native descriptor is ambiguous. Missing files, unreadable metadata, out-of-profile paths, architecture mismatch, or malformed and mismatched hashes make the entry ineligible. At menu startup, perform lightweight boot-ESP eligibility checks only (manifest schema, active slot, canonical native path, no duplicate native descriptor, `healthy == true`, valid 64-hex `sha256`, file existence and non-zero size, and native PE machine from the PE header prefix); do not eagerly hash foreign-architecture UKIs or block initial menu rendering on full-image hashing. Immediately before launching recovery, stream the native UKI once from a single open file handle in large sequential chunks (avoiding repeated reopen/seek calls), require `bytes_read == file_size`, and verify the full SHA-256 digest against `slot.sha256` before writing handoff or attempt state or calling `LoadImage`; fail closed on any I/O error, truncation, or digest mismatch. `healthy` is publisher eligibility and the digest is an integrity check. `verified_signature` must remain unknown (`None`) unless PE signature trust is independently established.

| Architecture | Firmware fallback path | Slot A UKI path | PE machine |
|---|---|---|---|
| x86_64 | \EFI\BOOT\BOOTX64.EFI | \EFI\sbm\recovery\slot-a\sbm-core-x86_64.efi | 0x8664 |
| AArch64 | \EFI\BOOT\BOOTAA64.EFI | \EFI\sbm\recovery\slot-a\sbm-core-aarch64.efi | 0xaa64 |

The picker exposes **Manage Computer** and selects the eligible native Slot A on the boot ESP. Report absence or validation failure in the picker. Slot B, cross-disk recovery and automatic rollback remain future profiles.

### Disk image and UKI

Use a protective MBR and GPT. In order, lay out the FAT32 ESP, x86_64 EROFS root, x86_64 verity hash, AArch64 EROFS root, and AArch64 verity hash. Preserve each mkosi data/hash partition's contents, architecture-specific DPS type GUID, unique PARTUUID and required attributes. Use one ESP of at least 512 MiB and retain at least 64 MiB of free space after placing both menu binaries, both UKIs, and the manifest.

Use a native mkosi UKI with the complete .cmdline, including its generated roothash. Verify during packaging that the first and last 128 bits of the hash resolve to that architecture's packaged data and hash PARTUUIDs, respectively. Linux must activate that dm-verity pair and mount the verified EROFS root. Preserve the embedded .cmdline when moving the UKI to its canonical EFI path.

Use firmware LoadImage via the exact ESP and file device path. Do not retry recovery through a pathless buffer load. Start the UKI without external EFI load options. When Secure Boot is enforced, firmware validates the PE image and systemd-stub uses its embedded .cmdline. External options cannot be relied on under Secure Boot when this section is present. Test signed recovery and rejection of untrusted images separately, with enrolled guest keys.

[Boot Loader Interface](https://systemd.io/BOOT_LOADER_INTERFACE/) · [Discoverable Partitions Specification](https://uapi-group.org/specifications/specs/discoverable_partitions_specification/) · [systemd-stub](https://www.freedesktop.org/software/systemd/man/latest/systemd-stub.html) · [systemd-veritysetup-generator](https://www.freedesktop.org/software/systemd/man/latest/systemd-veritysetup-generator.html)

## 10. Return path

Core emits the Spec 02 success report once its management UI is initialized and has received a daemon snapshot. Its Reboot control invokes an ordinary guest reboot. Firmware boots the same USB medium again through standard boot priority. The 0.0.1 picker does not consume \EFI\sbm\handoff\requested-command.json as a selection or power instruction.

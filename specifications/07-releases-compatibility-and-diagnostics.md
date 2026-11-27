# Spec 07: Release Manifests, Compatibility Negotiation & Diagnostics (`v1`)

**Status:** Normative Draft v1.0 (`sbm-types` + `sbm-protocol`, `#![no_std]` + `alloc`)

## 17. Artifact & Release Manifests (`ReleaseManifest`)

Publish trusted releases with TUF and Sigstore/Cosign verification. The 0.0.1 beta integration artifacts use the same manifest with hashes and an evidence receipt. Trust claims require a separate signed distribution gate.
- `schema_version: "1.0"` for the original format or `"1.1"` when publishing bootable media images
- `release_version: String` (SemVer, e.g. `"0.0.1"`, `"0.0.2"`, `"0.1.0"`)
- `channel: ReleaseChannel` (`Stable | Beta | Nightly`)
- `published_at_unix: i64`
- `git_commit: String`
- `minimum_protocol_major: u16` and `minimum_protocol_minor: u16`
- `artifacts: Vec<ReleaseArtifact>`:
  - `component: ComponentKind` (`SbmUefi | SbmCoreUki | SbmCoreSysext | SbmAsBootstrap | SbmInstaller | SbmServer | SbmAgent`)
  - `arch: CpuArch`
  - `platform: FirmwarePlatform`
  - `filename: String`
  - `size_bytes: u64`
  - `sha256: String`
  - `sbat_generation: Option<u32>` (for EFI binaries)
  - `signature_bundle_url: Option<String>`
- `media: Vec<ReleaseMedia>` (optional on the wire, defaults to empty)
  - `filename: String`
  - `size_bytes: u64`
  - `sha256: String`
  - `architectures: Vec<CpuArch>`

### Canonical Media Naming & Distribution Policy (SBM 0.0.1+)

Per [ADR-009](../architecture-decisions/ADR-009-dual-architecture-offline-preview-media.md), **architecture-specific bootable images** are the primary recommended release artifacts and are qualified independently:

- **Primary `x86_64` media:** `sbm-<version>-uefi-x86_64.raw` and `sbm-<version>-uefi-x86_64.raw.zst` (`media[].architectures: ["x86_64"]`), accompanied by a SHA-256 sidecar (`*.raw.zst.sha256`).
- **Primary `aarch64` media:** `sbm-<version>-uefi-aarch64.raw` and `sbm-<version>-uefi-aarch64.raw.zst` (`media[].architectures: ["aarch64"]`), accompanied by a SHA-256 sidecar (`*.raw.zst.sha256`).
- **Optional experimental dual-architecture media (non-gating):** `sbm-<version>-uefi-dualarch.raw` and `sbm-<version>-uefi-dualarch.raw.zst` (`media[].architectures: ["x86_64", "aarch64"]`). When built, it MUST be explicitly labeled experimental and MUST NOT block qualification or publication of either primary per-architecture release.

Each published manifest (`release-manifest.v1.json`) MUST populate `artifacts` with the exact final bytes of every included native firmware binary (`SbmUefi`) and recovery UKI (`SbmCoreUki`), and populate `media` with both raw and compressed forms of the published image(s). Digests are lowercase hexadecimal SHA-256. The companion `build-receipt.json` records source and dependency SHAs, builder tools and package lists, partition IDs and verity hashes, and exact build and verification commands.

### Optional Flasher & Provisioning Catalog Feed (`sbm-os-list.json`)

To enable direct reuse of existing cross-platform flashers prior to **`SBM Manager`** MVP (specifically **Raspberry Pi Imager `>= 2.0.11`** via `--repo` or **App Options -> Content Repository**; see [ADR-011](../architecture-decisions/ADR-011-native-preview-internal-install-and-media-creator.md) and [ADR-012](../architecture-decisions/ADR-012-profiles-network-recovery-modularity-and-ux.md)), releases MAY additionally publish a schema-validated `sbm-os-list.json` conforming to `rpi-imager`'s v4 OS list JSON schema (`url`, `image_download_size`, `image_download_sha256`, `extract_size`, `extract_sha256`, `architecture`, omitting `init_format`).

Readers with `1.0` support can continue consuming `1.0` manifests without `media`. Consumers that use `media` must validate the `1.1` schema version, media digest, and architecture list. The protocol version remains `1.0`.

## 17a. Modular Extension Descriptors (`ModuleDescriptor` & Signed `systemd-sysext`/`systemd-confext` DDIs, SBM 0.0.3–0.1+)

Per [ADR-012](../architecture-decisions/ADR-012-profiles-network-recovery-modularity-and-ux.md), optional SBM capabilities follow the 0SFS-style modularity philosophy using **`mkosi`** (`Format=sysext`/`Format=confext`, `Overlay=yes`, `BaseTrees=`), **`systemd-sysext`/`systemd-confext`**, signed `dm-verity` GPT DDIs, and **`systemd-sysupdate`**. SBM MUST NOT invent a general-purpose package manager.

### Signed Module Metadata (`ModuleDescriptor`)

Every optional capability module listed in a release catalog MUST be covered by signed TUF metadata and declare:

- `id: String` (stable lowercase identifier, e.g., `"gparted"`, `"fs-ntfs-btrfs"`, `"remote-recovery"`, `"wifi-firmware-extra"`, `"os-install-tools"`)
- `kind: ModuleKind` (`Sysext | Confext`)
- `version: String` (SemVer matching the built extension release)
- `sysext_level: String` (exact `SYSEXT_LEVEL` / `CONFEXT_LEVEL` compatibility token matching the target `sbm-core` base image ABI)
- `arch: CpuArch` (`x86_64 | aarch64`)
- `depends_on: Vec<String>` (flat, acyclic dependency list of other `ModuleDescriptor.id` values; cyclic graphs MUST be rejected fail-closed)
- `download_size_bytes: u64` and `uncompressed_size_bytes: u64`
- `min_ram_bytes: u64` (minimum RAM required to activate in `ram-only` mode)
- `sha256: String` (lowercase hex SHA-256 of the `.sysext.raw` / `.confext.raw` DDI image)
- `verity_roothash: String` (lowercase hex `dm-verity` root hash embedded in the signed DDI)
- `signature_bundle_url: Option<String>` (TUF / Cosign signature bundle in addition to the DDI's embedded PKCS#7 `usr-verity-sig` / `root-verity-sig` partition)
- `offline_profiles: Vec<String>` (which profiles pre-bundle this module by default, e.g., `["offline-full"]`, vs. on-demand only)

### Integrity, Immutability & Lifecycle Rules

1. **Signed-Only Production Enforcement:** Production `sbm-core` MUST enforce signed `dm-verity` DDIs via `systemd-sysext` / `systemd-confext` with `--image-policy='root=signed+absent:usr=signed+absent:=unused+absent'` (no unsigned or unverified extensions in production builds). Early boot/initrd extensions placed in `/loader/extensions/` measure into **TPM2 PCR 13**, preserving PCR 11 for the base UKI.
2. **Immutable Base Preservation:** Activating or deactivating modules uses OverlayFS merges over `/usr` (`sysext`) or `/etc` (`confext`) without modifying the verified `dm-verity` EROFS root filesystem.
3. **Full User Lifecycle:** Both **`SBM Manager`** (offline/USB provisioning) and **`sbm-core`** (runtime management) MUST support:
   - **Download:** Fetch over HTTPS (or `systemd-sysupdate --component=<id>`), verifying TUF metadata, SHA-256, `arch`, `sysext_level`, and acyclic `depends_on` closure before caching.
   - **Cache:** Store verified DDIs exclusively in RAM (`/run/extensions/` in `ram-only` mode) or on an explicitly authorized persistent cache medium (`/var/lib/extensions/` on `cached-usb` or `cached-ssd`—NEVER unauthorized internal disks).
   - **Activate:** Verify dependency closure and RAM budget, then invoke `systemd-sysext refresh` / `systemd-confext refresh` (reloading systemd units when `EXTENSION_RELOAD_MANAGER=1` is declared).
   - **Deactivate:** Refuse deactivation while a dependent active module requires it (or deactivate the dependent closure together with user confirmation), then unmerge cleanly.
   - **Remove:** Deactivate if active and delete the cached `.raw` DDI and sidecar metadata from the authorized cache store.

## 18. Compatibility Negotiation (`CapabilityHello`, `ProtocolVersion`)

Network sessions between SBM components exchange:
- `component: ComponentKind`
- `software_version: String`
- `supported_protocol_min: ProtocolVersion` (e.g., `1.0`)
- `supported_protocol_max: ProtocolVersion` (e.g., `1.0`)
- `features: Vec<String>` (feature capability flags, e.g. `["signed-commands-v1", "wayvnc-proxy", "boot-attempt-reporting", "ephemeral-recovery-v1", "sysext-modules-v1"]`)

**Negotiation Rule:** Both peers select the highest common `(major, minor)` where `major` matches. If no common major version exists, fail closed with `SbmErrorCode::IncompatibleProtocolVersion` and log an actionable upgrade message without executing any mutating command.

The offline EFI-to-Linux handoff checks its schema version through Spec 03 and does not open a network session.

## 19. Errors & Diagnostic Reporting (`SbmError`, `DiagnosticBundle`)

Structured, machine-readable, privacy-preserving errors shared across all crates (`no_std` compatible):
- `code: SbmErrorCode` (stable numeric + string enum: `InvalidSchema`, `SignatureInvalid`, `ReplayDetected`, `UnauthorizedScope`, `ProtectedPartitionBlocked`, `DiskIdentityMismatch`, `NetworkTimeout`, `TargetNotFound`, `ChainloadFailed`, `IncompatibleProtocolVersion`, `StorageIoError`, `UnsupportedPlatform`, `InternalError`)
- `component: ComponentKind`
- `recoverable: bool`
- `message: String` (concise human-readable summary; NEVER include secrets, passwords, BitLocker recovery keys, or raw private keys!)
- `context: Vec<(String, String)>` (bounded structured key-value pairs)
- `remediation_hint: Option<String>`

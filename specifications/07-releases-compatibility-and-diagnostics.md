# Spec 07: Release Manifests, Compatibility Negotiation & Diagnostics (`v1`)

**Status:** Normative Draft v1.0 (`sbm-types` + `sbm-protocol`, `#![no_std]` + `alloc`)

## 17. Artifact & Release Manifests (`ReleaseManifest`)

Published with every release in `sbm/releases/` and verified via TUF (`v1.0.33`) + Sigstore/Cosign:
- `schema_version: "1.0"`
- `release_version: String` (SemVer, e.g. `"0.1.0"`)
- `channel: ReleaseChannel` (`Stable | Beta | Nightly`)
- `published_at_unix: i64`
- `git_commit: String`
- `minimum_compatible_protocol: ProtocolVersion`
- `artifacts: Vec<ReleaseArtifact>`:
  - `component: ComponentKind` (`SbmUefi | SbmCoreUki | SbmCoreSysext | SbmAsBootstrap | SbmInstaller | SbmServer | SbmAgent`)
  - `arch: CpuArch`
  - `platform: FirmwarePlatform`
  - `filename: String`
  - `size_bytes: u64`
  - `sha256: String`
  - `sbat_generation: Option<u32>` (for EFI binaries)
  - `signature_bundle_url: Option<String>`

## 18. Compatibility Negotiation (`CapabilityHello`, `ProtocolVersion`)

All inter-component sessions (`sbm-uefi` <-> `sbm-server`, `sbm-core` <-> `sbm-server`, `sbm-agent` <-> `sbm-server`, `sbm-uefi` -> `sbm-core` handoff) exchange:
- `component: ComponentKind`
- `software_version: String`
- `supported_protocol_min: ProtocolVersion` (e.g., `1.0`)
- `supported_protocol_max: ProtocolVersion` (e.g., `1.0`)
- `features: Vec<String>` (feature capability flags, e.g. `["signed-commands-v1", "wayvnc-proxy", "boot-attempt-reporting"]`)

**Negotiation Rule:** Both peers select the highest common `(major, minor)` where `major` matches. If no common major version exists, fail closed with `SbmErrorCode::IncompatibleProtocolVersion` and log an actionable upgrade message without executing any mutating command.

## 19. Errors & Diagnostic Reporting (`SbmError`, `DiagnosticBundle`)

Structured, machine-readable, privacy-preserving errors shared across all crates (`no_std` compatible):
- `code: SbmErrorCode` (stable numeric + string enum: `InvalidSchema`, `SignatureInvalid`, `ReplayDetected`, `UnauthorizedScope`, `ProtectedPartitionBlocked`, `DiskIdentityMismatch`, `NetworkTimeout`, `TargetNotFound`, `ChainloadFailed`, `IncompatibleProtocolVersion`, `StorageIoError`, `UnsupportedPlatform`, `InternalError`)
- `component: ComponentKind`
- `recoverable: bool`
- `message: String` (concise human-readable summary; NEVER include secrets, passwords, BitLocker recovery keys, or raw private keys!)
- `context: Vec<(String, String)>` (bounded structured key-value pairs)
- `remediation_hint: Option<String>`

# Spec 07: Release Manifests, Compatibility Negotiation & Diagnostics (`v1`)

**Status:** Normative Draft v1.0 (`sbm-types` + `sbm-protocol`, `#![no_std]` + `alloc`)

## 17. Artifact & Release Manifests (`ReleaseManifest`)

Publish trusted releases with TUF and Sigstore/Cosign verification. The 0.0.1 beta integration artifact uses the same manifest with hashes and an evidence receipt. Trust claims require a separate signed distribution gate.
- `schema_version: "1.0"` for the original format or `"1.1"` when publishing combined boot media
- `release_version: String` (SemVer, e.g. `"0.1.0"`)
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

The 0.0.1 manifest lists both native firmware binaries and both UKIs as component artifacts using their final bytes. It lists both compressed and raw forms of the same image under `media`, with architectures `["x86_64", "aarch64"]`. Use `sbm-0.0.1-uefi-dualarch.raw` and `sbm-0.0.1-uefi-dualarch.raw.zst` as filenames and publish a SHA-256 sidecar for the compressed image. Store the manifest as `release-manifest.v1.json`. Digests are lowercase hexadecimal SHA-256. The build receipt records source and dependency SHAs, tools and packages, partition IDs and hashes, and the build and verification commands.

Readers with 1.0 support can continue consuming 1.0 manifests without `media`. Consumers that need combined media must add explicit 1.1 support and validate the media digest and architecture list. The protocol version remains 1.0.

## 18. Compatibility Negotiation (`CapabilityHello`, `ProtocolVersion`)

Network sessions between SBM components exchange:
- `component: ComponentKind`
- `software_version: String`
- `supported_protocol_min: ProtocolVersion` (e.g., `1.0`)
- `supported_protocol_max: ProtocolVersion` (e.g., `1.0`)
- `features: Vec<String>` (feature capability flags, e.g. `["signed-commands-v1", "wayvnc-proxy", "boot-attempt-reporting"]`)

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

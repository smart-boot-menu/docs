# Spec 05: Operating System Catalog Definitions (`v1`)

**Status:** Normative Draft v1.0 (`sbm-types`, `sbm-schema`, `sbm-catalog`)

## 13. Catalog Structure & Verification (`CatalogIndex`, `OsCatalogEntry`)

`sbm-catalog` publishes a **TUF-signed (`v1.0.33`)** catalog index + per-OS JSON/TOML definitions that can be consumed online OR mirrored locally for 100% offline air-gapped operation.

### `OsCatalogEntry` Schema
- `schema_version: "1.0"`
- `id: String` (stable reverse-DNS or slug, e.g. `"fedora-workstation"`, `"debian"`, `"ubuntu-desktop"`, `"arch-linux"`, `"freebsd"`, `"windows-11-bring-your-own-iso"`)
- `name: String`, `os_family: OsFamily`, `vendor: String`, `homepage_url: String`
- `releases: Vec<OsReleaseEntry>`:
  - `version: String` (e.g., `"43"`, `"13.2"`)
  - `channel: ReleaseChannel` (`Stable | Lts | Rolling | Testing`)
  - `eol_date: Option<String>` (ISO-8601 date)
  - `architectures: Vec<CpuArch>`
  - `supported_platforms: Vec<FirmwarePlatform>` (explicitly marks whether Apple Silicon `linux-asahi` is supported so incompatible generic ARM64 images are NEVER offered on Macs!)
  - `min_ram_mib: u32`, `min_disk_gib: u32`
  - `install_strategy: InstallStrategy`
  - `artifacts: Vec<CatalogArtifact>`:
    - `filename: String`
    - `size_bytes: u64`
    - `sha256: String` (mandatory; MD5/SHA-1 forbidden)
    - `sha512: Option<String>`
    - `upstream_pgp_or_minisign_url: Option<String>`
    - `mirrors: Vec<String>` (HTTPS only; HTTP forbidden unless checksum + signature verified)
    - `user_supplied_only: bool` (e.g., for proprietary/EULA-gated ISOs like Windows 11 where the user supplies or downloads from Microsoft directly and SBM verifies integrity)
  - `recipe_version: String`

### Security Invariants
- **NO unsigned executable scripts in catalog entries:** Installation recipes are **declarative** (referencing built-in, audited `sbm-core` install strategies), NEVER arbitrary `curl | sh` scripts downloaded from the catalog!
- Every downloaded OS artifact MUST pass exact `sha256` verification against TUF-signed catalog metadata before any write or boot occurs.

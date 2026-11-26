# Smart Boot Menu — Threat Model & Safety Invariants

**Version:** 1.0-alpha (2026-11-26)  
**Supporting Research:** `research/architecture/security-and-supply-chain.md`, `research/hardware/apple-silicon-m1-to-m5.md`

Smart Boot Menu executes at **Ring 0 / pre-OS UEFI** (`sbm-uefi`) and **privileged Linux root** (`sbm-core`). A compromise of SBM compromises every OS and unencrypted disk attached to the machine. Security and data safety are foundational architectural constraints, not post-hoc features.

---

## 1. Trust Boundaries

1. **TB-1 — Firmware / Hardware Root of Trust -> `sbm-uefi`:** UEFI Secure Boot (`PK`/`KEK`/`db`/`dbx`, `shim 16.1`, Microsoft UEFI CA 2023) or Apple Silicon Boot ROM -> LLB -> `LocalPolicy` -> `m1n1` -> U-Boot.
2. **TB-2 — Local Physical User vs Remote Network (`sbm-server`):** Physical keyboard/power-button access always supersedes remote boot commands. Remote channels are untrusted by default until explicit device enrollment.
3. **TB-3 — `sbm-uefi` (Pre-OS Fast Path) -> `sbm-core` (Management Path):** `sbm-uefi` NEVER executes arbitrary remote commands, shells, or disk mutations. Complex operations run only inside signed, `dm-verity`-verified `sbm-core`.
4. **TB-4 — `sbm-core` Orchestrator -> External Storage & OS Media:** All downloaded OS images and catalogs are untrusted until SHA-256 + TUF signature verification succeeds; all attached disks are read-only by default.

---

## 2. Threat Matrix (12 Required Threat Classes)

| # | Threat | Attack Vector / Failure Mode | Architectural Mitigations | Residual Risk / Honest Limits |
|---|---|---|---|---|
| **T1** | **Malicious Remote Commands** | Attacker with stolen session or MitM sends wipe/boot command | Signed `CommandEnvelope` (Ed25519), strict `AuthScope` separation, Passkey step-up for `storage_destructive`, **no RCE or disk ops in `sbm-uefi`** | Compromised operator Passkey + `storage_destructive` role can authorize valid destructive commands; mitigate via audit logs & optional physical confirmation mode |
| **T2** | **Compromised `sbm-server`** | Server host is breached; attacker attempts fleet-wide compromise | Offline local override always works; `sbm-uefi` only boots known local targets (no remote arbitrary EFI exec); optional physical-presence-required flag for destructive ops; separate offline root keys for TUF catalog/releases | A compromised server holding the live `server_command_verify_key` can send valid `BootSelect` commands unless hardware-backed operator signing keys are enabled in Phase 2 |
| **T3** | **Compromised Software Updates** | Mirror hijacking, rollback to vulnerable EFI binary, freeze attack | **TUF v1.0.33** metadata (offline root key, timestamp/snapshot/targets roles), `dm-verity` for `sbm-core`, **SBAT** revocation metadata in `sbm-uefi`, Sigstore/SLSA CI provenance, reproducible builds | Compromise of release build pipeline mitigated by reproducible builds + offline TUF root threshold signing |
| **T4** | **Malicious Catalog Entries** | Trojanized OS ISO URL or malicious install script in `sbm-catalog` | TUF-signed `sbm-catalog` metadata, mandatory SHA-256 verification of all artifacts, **declarative-only recipes (zero arbitrary shell scripts)**, upstream PGP/SHA verification in CI | Upstream distro mirror compromise before catalog update if upstream itself is breached |
| **T5** | **Unauthorized Disk Access** | Unintended read/write to BitLocker, LUKS, macOS, or other OS volumes | Read-only discovery by default; `PartitionProtectionClass` hard-blocks Apple `iSCPreboot`/`Recovery`/macOS and BitLocker volumes; stable `DiskIdentity` matching | Root inside `sbm-core` interactive shell has physical-equivalent access; protect `sbm-core` entry with optional PIN/password when remote/kiosk locked |
| **T6** | **Credential Theft** | Stolen operator passwords or device private keys | Passwordless **WebAuthn/Passkeys** (phishing-resistant), short-lived enrollment tokens, TPM 2.0 sealed device keys (`tss-esapi`) where available, zero plaintext secrets in ESP handoff or logs | On hardware without TPM 2.0, device key on ESP/disk is readable by physical attacker with disk access |
| **T7** | **Replay Attacks** | Re-sending an old `BootRecovery` or `Format` command envelope | Strictly monotonic `sequence` persisted atomically per device + `device_id` binding + short `expires_at_unix` TTL + `one_shot` consumed-before-boot flag | Clockless UEFI systems without RTC require online challenge nonce before trusting TTL |
| **T8** | **Boot-Policy Manipulation** | Local malware or attacker silently alters `\EFI\sbm\policy.json` | Optional signed policy verification; Secure Boot protects `sbm-uefi` & `sbm-core` binaries; TPM Measured Boot records PCR 4/7/11; BitLocker warning before `BootOrder` edits | An attacker who already owns Administrator/root on an installed OS can write to the FAT32 ESP; full tamper-evidence requires TPM/Secure Boot + signed policies |
| **T9** | **EFI Persistence Attacks** | Bootkit replaces `sbm-uefi.efi` or injects rogue EFI driver | UEFI Secure Boot (`shim 16.1` + Microsoft UEFI CA 2023 / custom PK/KEK/db), `MOK_POLICY_REQUIRE_NX`, no auto-loading of unsigned third-party `.efi` drivers | If Secure Boot is disabled by the user, firmware cannot block unsigned EFI replacements |
| **T10** | **Physical Access** | Evil-maid attacker reboots unattended laptop into `sbm-core` | Optional local admin PIN/lockout for entering `sbm-core` or editing menu; full-disk encryption (LUKS2/BitLocker/FileVault) remains the primary defense against physical disk access | Physical access to unlocked firmware setup allows booting external media unless firmware password + FDE are enabled |
| **T11** | **Supply-Chain Attacks** | Compromised Rust crate, C recovery tool, or CI build runner | Pinned `Cargo.lock`, `cargo-audit` / `cargo-deny` in CI, zero opaque binary blobs (no Ventoy blobs!), curated upstream Fedora/Asahi RPMs, reproducible builds + SLSA Level 3 provenance | Ongoing dependency hygiene required across every release |
| **T12** | **Accidental Destructive Operations** | User picks wrong disk or power fails mid-resize | Stable `DiskIdentity` (never `/dev/sdX`), expected partition table hash lock, human-readable dry-run diff, explicit destructive confirmation, platform-critical partition guardrails, RAM-resident `sbm-core` so target disks are unmounted cleanly | **Honest Disclaimer:** Formatting, deleting, overwriting, or power-interrupted filesystem resizes are **NOT transactional or reversible**. SBM MUST explicitly warn users to back up critical data before destructive operations. |

---

## 3. Mandatory Security & Safety Invariants (Non-Negotiable)

1. **Zero mandatory telemetry, zero cloud accounts, 100% offline local utility.**
2. **Zero arbitrary remote code execution in `sbm-uefi`.**
3. **BitLocker Safety Gate:** `install` MUST detect BitLocker volumes and require explicit user confirmation of BitLocker Recovery Key backup (or 1-reboot BitLocker suspension) before altering UEFI `BootOrder` or Secure Boot keys.
4. **Apple Silicon Safety Gate:** Never modify Apple `iSCPreboot`, `RecoveryOS`, or macOS APFS containers from `sbm-core` or `install` without dedicated, verified Apple-safe tooling (and never automatically).
5. **Bounded Network Timeouts:** `sbm-uefi` network operations MUST never hang boot (hard cap ≤ 4–5 seconds, instantly skippable by keypress).

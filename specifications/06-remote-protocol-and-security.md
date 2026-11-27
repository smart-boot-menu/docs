# Spec 06: Remote Enrollment, AuthN/AuthZ & Signed Command Envelopes (`v1`)

**Status:** Normative Draft v1.0 (`sbm-protocol`, `#![no_std]` + `alloc`)

## 14. Remote Device Enrollment (`EnrollmentRequest`, `EnrollmentResponse`)

Remote management is **disabled by default**. Enrollment requires explicit physical or authenticated local action:
1. Operator generates a short-lived, single-use **Enrollment Token** on `sbm-server` (expires in ≤ 15 minutes; bound to server CA fingerprint `sha256:...`).
2. Device (`sbm-core` or `install`) generates a fresh **Ed25519 keypair** (TPM-sealed if available) and sends `EnrollmentRequest` over TLS pinned to the server CA fingerprint:
   - `enrollment_token: String`
   - `device_public_key_ed25519: [u8; 32]` (hex/base64 on wire)
   - `hostname: String`
   - `platform: PlatformCapabilities`
   - `tpm_attestation: Option<Vec<u8>>`
3. Server validates token, allocates a stable `device_id` (UUIDv7), signs a device X.509 mTLS certificate (`rcgen`) + returns `EnrollmentResponse` containing `device_id`, `client_cert_pem`, `server_command_verify_key_ed25519`, and `granted_capabilities`.

## 15. Authentication & Authorization (`AuthScope`)

Strict least-privilege separation between read-only, boot-selection, recovery, and destructive storage operations:

| Scope | Wire String | Permitted Actions | Allowed in `sbm-uefi`? |
|---|---|---|---|
| `ReadOnly` | `"read_only"` | View device inventory, health, boot targets, logs | Yes |
| `BootSelect` | `"boot_select"` | Set one-shot or default boot target (from known targets) | **Yes (only mutating remote scope allowed in UEFI)** |
| `RecoveryAccess` | `"recovery_access"` | Boot `sbm-core` from the management OS, open remote desktop/terminal, read-only diagnostics | **NO.** `sbm-uefi` accepts only `read_only` and `boot_select` (`AuthScope::is_allowed_in_uefi`, §16 rule 5). A recovery target is selected with `BootSelect` / `BootTarget`. `BootRecovery` is not on the UEFI boot-policy poll. |
| `OsInstall` | `"os_install"` | Execute verified `OsInstallPlan` inside `sbm-core` | **NO** |
| `StorageDestructive` | `"storage_destructive"` | Execute destructive `StorageOperationPlan` inside `sbm-core` (requires step-up Passkey confirmation) | **NO** |
| `Admin` | `"admin"` | Enroll/revoke devices, manage keys, rotate server signing key | **NO** |

UEFI may select a discovered recovery target with the same `boot_select` mechanism used for other discovered targets. `BootRecovery` under `recovery_access` is for the Core environment. The 0.0.1 profile keeps remote execution disabled.

## 16. Signed Remote Command Envelopes (`CommandEnvelope`)

Every remote instruction sent from `sbm-server` to `sbm-uefi`, `sbm-core`, or `sbm-agent` MUST be wrapped in a canonically serialized `CommandEnvelope`:
- `schema_version: "1.0"`
- `command_id: String` (UUIDv7)
- `device_id: String` (prevents cross-device replay!)
- `sequence: u64` (strictly monotonically increasing per device; device persists `last_seen_sequence` atomically on ESP and rejects any `sequence <= last_seen_sequence`!)
- `issued_at_unix: i64`
- `expires_at_unix: i64` (short TTL; max 24h for boot commands, max 15m for destructive storage plans)
- `required_scope: AuthScope`
- `issuer_key_id: String`
- `payload: CommandPayload` (e.g., `BootCommand`, `RunStoragePlan`, `RunOsInstall`, `RevokeEnrollment`)
- `signature_ed25519_hex: String` (lowercase hex Ed25519 signature; the prose name `signature_ed25519` in earlier drafts is this wire field, matching `sbm-protocol` and `test-vectors/command-envelope.v1.json`)

#### 16.1 Command Signing Profile v1 (normative)

The signature input is exactly:

```text
b"sbm-cmd-v1\0" || canonical_json
```

`COMMAND_SIGNING_DOMAIN_V1` in `sbm-protocol` is the domain prefix. `canonical_json` is the compact UTF-8 JSON object of every envelope field **except** `signature_ed25519_hex`, with no insignificant whitespace and with keys in this order:

1. `schema_version`
2. `command_id`
3. `device_id`
4. `sequence` (decimal integer)
5. `issued_at_unix` (decimal integer)
6. `expires_at_unix` (decimal integer)
7. `required_scope`
8. `issuer_key_id`
9. `payload`

`payload` uses the `sbm-protocol` 0.1 serde representation: internally tagged enums put the tag field first (`kind`, then `action` / `mode`), then variant fields in declaration order. Omitted optional fields are absent. Strings use standard JSON escaping. This profile is the serde_json compact form of those types, not RFC 8785. `sbm_protocol::command_signing_preimage` is that byte string, and `sbm-server` signs it. `test-vectors/command-signing.v1.json` publishes a test seed (not a production key) and the signatures for a boot selection, a recovery command with escaped text, and a revocation. A device persists `last_seen_sequence` only after the signature, `device_id`, and expiry checks succeed, including when it then rejects the command for scope. Invalid signatures must not advance the counter.

### Verification Rules (Enforced by `sbm-uefi`, `sbm-core`, `sbm-agent`)
1. Verify Ed25519 signature against pinned `server_command_verify_key_ed25519`.
2. Verify `device_id == local_device_id`.
3. Verify `sequence > persisted_last_sequence` (anti-replay).
4. Verify `now <= expires_at_unix` (when trusted RTC/clock is available; if UEFI RTC is known-unset, require short-lived online nonce challenge before accepting).
5. Verify scope, action and target against local policy. UEFI handles read-only reporting and accepts only `boot_select` with a known `boot_target` for remote execution. A permitted scope does not authorize other payloads.

## 16a. Ephemeral Remote Recovery Sessions (`SBM Manager` Recovery USB, SBM 0.1+)

Per [ADR-012](../architecture-decisions/ADR-012-profiles-network-recovery-modularity-and-ux.md), a primary SBM use case is preparing a recovery USB with **`SBM Manager`** on a working computer to remotely repair a target computer whose installed OS is completely broken. Because a recovery USB is removable physical media that can be lost, lent, or reused, ephemeral remote recovery sessions MUST enforce stricter rules than persistent fleet enrollment (§14):

1. **Opt-In Only:** Remote access is **disabled by default** on every recovery USB unless the owner explicitly enables an ephemeral remote-recovery session in `SBM Manager` (or locally inside `sbm-core`).
2. **Zero Permanent Shared Credentials on USB:** A recovery USB MUST NEVER store long-lived admin private keys, reusable fleet credentials, static SSH passwords, or permanent shared secrets.
3. **Zero Unattended Arbitrary Command Execution:** Recovery USBs MUST NOT carry auto-executing scripts or unattended root command hooks. Any remote session connects only inside authenticated `sbm-core` (never inside `sbm-uefi`) and is restricted to the explicitly authorized session scopes.
4. **Ephemeral Session Grant (`EphemeralRecoveryGrant`):** When enabled in `SBM Manager`, the USB stores only a time-bounded, single-session grant containing:
   - `session_id` (UUIDv7)
   - `rendezvous_endpoint` + pinned server/peer CA or WireGuard/Ed25519 public key fingerprint
   - `one_time_bootstrap_token` (short-lived, single-use, invalidated upon first handshake or expiry)
   - `expires_at_unix` (explicit short TTL, e.g., hours—never indefinite)
   - `allowed_scopes` (explicit least-privilege subset, e.g., `recovery_access`, optionally `storage_destructive` with explicit step-up confirmation)
5. **End-to-End Authenticated & Encrypted Transport:** On boot into verified `sbm-core`, the target generates a **fresh in-RAM ephemeral keypair** (never written to internal SSDs), authenticates the pinned owner/relay public key, establishes an end-to-end encrypted session (e.g., WireGuard / mTLS + WayVNC/PTY proxy), and binds all interactive or command actions to that single session.
6. **Local Visibility & Immediate Revocation:**
   - The local display MUST show an unmistakable active-session banner (peer fingerprint / session ID, elapsed time, and granted scope) whenever a remote recovery session is active or awaiting connection.
   - A single local keypress/UI button (**Revoke & Disconnect**) MUST immediately terminate the session and wipe the in-RAM ephemeral session keys.
   - The owner MAY revoke the grant remotely at any time from `SBM Manager` or `sbm-server`; the session automatically expires and revokes on exit, TTL expiry, or reboot.

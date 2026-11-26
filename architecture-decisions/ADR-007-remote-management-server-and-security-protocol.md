# ADR-007: Self-Hosted Server Stack, Cryptographic Identity & Remote Protocol

- **Status:** Accepted (SBM 0.1 Baseline)
- **Date:** 2026-11-26
- **Supporting Research:** `research/ecosystem/remote-management.md`, `research/architecture/security-and-supply-chain.md`

## Context

SBM supports optional self-hosted remote management (`sbm-server`) while guaranteeing 100% offline local operation without any server or account. Remote commands can reboot machines, select OSes, or trigger destructive storage operations, making authentication, authorization, and replay protection critical.

## Decision

1. **Server Backend Stack:** Rust (`axum 0.8.9` + `tokio 1.54.1` + `sqlx 0.9.1` + `rustls >= 0.23.46` + `rcgen 0.14.10`).
2. **Database:** **SQLite (WAL mode)** as the zero-setup default (single binary + single database file for home/homelab self-hosting); **PostgreSQL** supported via `sqlx` for larger multi-operator deployments.
3. **Web Dashboard Frontend:** **SvelteKit (`3.4.0`, TypeScript)** embedded as static assets in `sbm-server`, integrating **`@novnc/novnc` (`MPL-2.0`)** for browser-native remote desktop (backed by `wayvnc` in `sbm-core`) and **`xterm.js`** for browser-native remote terminal — avoiding `AGPL-3.0` coupling with RustDesk while allowing optional external RustDesk / PiKVM / JetKVM / Redfish integrations.
4. **Operator AuthN/AuthZ:** **Passkeys / WebAuthn (`webauthn-rs 0.6.2`)** as primary passwordless auth + strict RBAC scopes (`read_only`, `boot_select`, `recovery_access`, `storage_destructive`, `admin`) — destructive operations require a distinct scope and step-up verification.
5. **Device Enrollment & Signed Command Envelopes:**
   - Explicit opt-in enrollment via short-lived one-time enrollment token.
   - Per-device **Ed25519 identity** (TPM 2.0 backed via `tss-esapi` where available) + **mTLS client certificate**.
   - Every remote command is a canonical **signed `CommandEnvelope`** (`sbm-protocol`) with `command_id` (UUIDv7), `device_id`, `issued_at`, `expires_at`, strictly monotonic `sequence` number, `scope`, and Ed25519 signature -> end-to-end integrity and replay protection even across reverse proxies.
6. **Transport Evolution:** HTTPS (TLS 1.3) + WebSockets (outbound device-initiated) for SBM 0.1; evaluate **Iroh (`MIT OR Apache-2.0`, QUIC P2P/NAT traversal)** and **WireGuard** as optional Phase 2 transports.

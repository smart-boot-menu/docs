# Spec 02: Boot Policies, Commands & Attempt Reporting (`v1`)

**Status:** Normative Draft v1.0 (`sbm-types` + `sbm-protocol`, `#![no_std]` + `alloc`)

## 5. Boot Commands (`BootCommand`)

A `BootCommand` represents an explicit local or remote instruction to select what executes next:
- `BootTarget { target_id: BootTargetId, one_shot: bool, extra_cmdline_preset: Option<String> }`
  - *Security constraint:* `sbm-uefi` MUST NOT accept arbitrary untrusted kernel command-line strings remotely; only allowlisted preset names or signed target IDs are permitted in UEFI.
- `BootRecovery { slot: RecoverySlotPreference, mode: RecoveryLaunchMode }` (`Interactive | RemoteAssist | DiagnosticOnly | PlannedOperation { plan_id }`)
- `RebootToFirmwareSetup`
- `Shutdown` / `Reboot`

## 6. Boot Policies (`BootPolicy`)

Stored atomically on the ESP (`\EFI\sbm\policy.json`) and optionally refreshed from `sbm-server`:
- `schema_version: String` (`"1.0"`)
- `policy_revision: u64`
- `default_target: Option<BootTargetId>`
- `fallback_target: Option<BootTargetId>` (defaults to `sbm-core` recovery if omitted)
- `timeout_seconds: u16` (`0` = immediate boot unless hotkey held; `65535` = wait indefinitely)
- `one_shot: Option<OneShotBoot>` (`target_id`, `command_id`, `expires_at_unix`, `consumed: bool`)
- `max_failed_attempts: u8` (default `3` before automatic fallback)
- `network_policy_timeout_ms: u32` (default `3000`, hard-capped at `5000` in `sbm-uefi`)
- `allow_remote_boot_select: bool` (default `false` until explicitly enrolled!)
- `offline_behavior: OfflineBehavior` (`UseLocalPolicy | WaitForUser`)

### Resolution Precedence (Deterministic)
1. **Explicit local physical hotkey/user selection** ALWAYS overrides remote or default policy (local owner supremacy).
2. **Valid, unconsumed, unexpired `one_shot` command** (local or verified remote) — marked `consumed = true` atomically *before* calling `StartImage`.
3. **Healthy `default_target`** (if `consecutive_failures < max_failed_attempts`).
4. **`fallback_target`** or interactive graphical menu if default failed or is missing.

## 7. Boot Attempt & Success Reporting (`BootAttemptRecord`, `BootSuccessReport`)

Enables automatic rollback for OS updates and `sbm-core` A/B slots:
- **Before `StartImage` (`sbm-uefi`):** Writes/updates `\EFI\sbm\state\last_attempt.json`:
  - `attempt_id: String` (UUIDv7 or monotonic counter + boot nonce)
  - `target_id: BootTargetId`
  - `slot: Option<String>`
  - `timestamp_unix: Option<i64>`
  - `consecutive_failures: u8` (incremented pre-boot)
  - `status: BootAttemptStatus` (`Started | Confirmed | Failed | RolledBack`)
- **After OS / `sbm-core` boots cleanly (`sbm-agent` or `sbm-core`):** Resets `consecutive_failures = 0`, sets `status = Confirmed`, and emits `BootSuccessReport` locally (to ESP state) and optionally to `sbm-server`.

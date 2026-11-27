# Spec 02: Boot Policies, Commands & Attempt Reporting (`v1`)

**Status:** Normative Draft v1.0 (`sbm-types` + `sbm-protocol`, `#![no_std]` + `alloc`)

## 5. Boot Commands (`BootCommand`)

A `BootCommand` represents an explicit local or remote instruction to select what executes next:
- `BootTarget { target_id: BootTargetId, one_shot: bool, extra_cmdline_preset: Option<String> }`
  - *Security constraint:* `sbm-uefi` MUST NOT accept arbitrary untrusted kernel command-line strings remotely; only allowlisted preset names or signed target IDs are permitted in UEFI.
  - *Canonical preset allowlist (`v1`):* `quiet`, `verbose`, `nomodeset`, and `diagnostic`. Both `sbm-server` and `sbm-uefi` MUST enforce this exact set and reject unknown presets.
- `BootRecovery { slot: RecoverySlotPreference, mode: RecoveryLaunchMode }` (`Interactive | RemoteAssist | DiagnosticOnly | PlannedOperation { plan_id }`)
- `RebootToFirmwareSetup`
- `Shutdown` / `Reboot`

## 5a. Local One-Shot Boot Request from SBM Core (`LocalBootRequest`, 0.0.1)

SBM Core asks the picker to start one discovered boot target on the next start by writing a `LocalBootRequest` to `\EFI\sbm\handoff\requested-command.json` on the boot ESP and then rebooting normally.

| Field | Value |
|---|---|
| `schema_version` | `1.0` |
| `request_id` | Fresh UUIDv4 |
| `attempt_id` | `attempt_id` of the verified integrated handoff Core is running under (Spec 03 §8) |
| `requested_by` | `sbm-core-uki` |
| `requested_at_unix` | Core's clock when the user chose the target |
| `command` | `BootTarget { target_id, one_shot: true, extra_cmdline_preset }`; the preset is absent or one of the §5 allowlist |

**Producer (Core).** Offer and accept only targets listed in the verified integrated handoff's `discovered_targets`, excluding `sbm_core` entries. Never write an installed-OS identifier (`os:<partuuid>`, Spec 01 §4) or a target inferred from one. Without a verified integrated handoff, offer no boot targets. Write a temporary file, `fsync` it, rename it over the destination, and sync the directory.

**Consumer (picker).** Read the request only from the boot ESP; requests on other ESPs are never read, acted on, or deleted. Delete the file and confirm the deletion before acting on it; if it cannot be deleted, do not act on it, because it would replay on the next start. Then honour it only if every check passes:

1. It decodes as a `LocalBootRequest` with schema `1.0`, `requested_by == sbm-core-uki`, and a UUIDv4 `request_id`. A bare `BootCommand` written by an older Core is incompatible and is rejected.
2. `attempt_id` equals the current `last_attempt.json` record, that record's target is `sbm-core:slot-a`, and its status is `confirmed` after the Spec 02 §7 success report has been applied. A request from an earlier, unconfirmed, or different attempt is stale.
3. `command` is `BootTarget` with `one_shot == true` and an absent or allowlisted preset. Power, firmware-setup and recovery commands are not accepted from this file.
4. Exactly one current discovered target has `target_id`, and it is selectable (visible, eligible, and not known to be rejected).

A valid request boots its target immediately, ahead of the default and fallback (§6 precedence 2), unless a local key is held. Any failed check leaves the picker in the interactive menu with no countdown and an on-screen explanation naming the problem (for example, the target is no longer present). The picker never substitutes another entry for a request it cannot honour.

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
2. **Valid, unconsumed, unexpired `one_shot` command** (a §5a local request, or a policy `one_shot` from a verified remote) — consumed durably *before* calling `StartImage` (the §5a file is deleted; a policy `one_shot` is persisted with `consumed = true`). If consumption cannot be persisted, the picker does not start the target. A `one_shot` whose target is missing or not selectable produces the interactive menu with an explanation, never a different target.
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
- After the OS boots cleanly, its reporter writes `BootSuccessReport` to the ESP. UEFI owns the attempt record and applies confirmation on its next start.

### 0.0.1 recovery profile

Create a fresh UUIDv4 for each recovery attempt before launch. Use `EFI_RNG_PROTOCOL` when firmware provides it. Otherwise derive the 122 random bits from SHA-256 over the firmware monotonic count, the RTC time, the CPU cycle counter, and the previous attempt ID; the result must differ from the previous attempt ID. Attempt IDs are correlation identifiers, not secrets. Increment the prior failure count only for an unconfirmed attempt for the same target; an attempt for a different target starts at one. Persist and flush `last_attempt.json` with status `started` before launch; if that fails, do not start recovery. Flush temporary data before publishing it: write `<name>.tmp`, flush it, remove the old file, rename the temporary file in place, and flush again. If the record is missing but a complete `.tmp` exists, use the `.tmp` record; treat interrupted or malformed FAT records as unconfirmed. If recovery `StartImage` returns, mark the attempt failed and show the picker.

Core writes and syncs `\EFI\sbm\state\boot-success.json` on the same ESP after the UI receives a live daemon snapshot and the Spec 03 handoff validates. The report contains `schema_version`, `attempt_id`, `target_id`, `confirmed_by`, and `confirmed_at_unix`. Read the report only from the boot ESP. Require schema `1.0`, a current `started` record, matching IDs, target `sbm-core:slot-a`, and reporter `sbm-core-uki` before setting `confirmed` and resetting the count. There is no wildcard attempt ID. A stale report cannot confirm a later attempt and is deleted. Confirming an attempt twice is idempotent.

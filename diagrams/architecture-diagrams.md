# Smart Boot Menu — System Architecture & Workflow Diagrams

**Last updated:** 2026-11-26  
All 10 canonical architecture, dependency, boot, installation, security, and storage lifecycle diagrams for Smart Boot Menu.

---

## 1. Overall Architecture

```mermaid
flowchart TB
    subgraph Hardware["Physical / Virtual Hardware"]
        HW_X86["x86-64 / ARM64 UEFI Firmware"]
        HW_AS["Apple Silicon (Boot ROM -> LLB -> iBoot)"]
        HW_DISKS["Internal & External Storage (NVMe / SATA / USB)"]
    end

    subgraph AppleBootstrap["Apple Silicon Bootstrap (sbm-as)"]
        M1N1["m1n1 Stage 1/2"]
        UBOOT["Asahi U-Boot (CONFIG_EFI_LOADER)"]
    end

    subgraph FastPath["Fast Path — Pre-OS (sbm-uefi)"]
        UEFI["sbm-uefi (Native Rust UEFI App)"]
        UEFI_UI["Software GOP Renderer + Input"]
        UEFI_DISC["OS & BLS/UKI Discovery"]
        UEFI_NET["Bounded Policy Client (Read-Only)"]
    end

    subgraph MgmtPath["Management Path — Linux (sbm-core)"]
        CORE_BOOT["Signed UKI + dm-verity EROFS"]
        CORE_GUI["labwc Wayland + XWayland + Iced UI + GParted"]
        CORE_ENG["Rust Storage, OS Installer & Recovery Engine"]
        CORE_REM["wayvnc + PTY Remote Bridge"]
    end

    subgraph InstalledOS["Installed Operating Systems"]
        OS_WIN["Windows"]
        OS_LIN["Linux / BSD"]
        OS_MAC["macOS (Apple Silicon Reboot Target)"]
        AGENT["Optional sbm-agent Daemon"]
    end

    subgraph OptionalRemote["Optional Self-Hosted Services"]
        SERVER["sbm-server (Axum + SQLite/PG + SvelteKit + noVNC)"]
        CATALOG["sbm-catalog (TUF-Signed OS Definitions)"]
    end

    HW_X86 --> UEFI
    HW_AS --> M1N1 --> UBOOT --> UEFI
    UEFI --- UEFI_UI
    UEFI --- UEFI_DISC
    UEFI --- UEFI_NET
    UEFI -->|"EFI Chainload (Normal Fast Boot)"| OS_WIN
    UEFI -->|"EFI Chainload / BLS"| OS_LIN
    UEFI -.->|"Apple Reboot Handoff"| OS_MAC
    UEFI -->|"UKI Launch + Handoff"| CORE_BOOT
    CORE_BOOT --> CORE_GUI
    CORE_BOOT --> CORE_ENG
    CORE_BOOT --> CORE_REM
    CORE_ENG -->|"Safe Plan Execution"| HW_DISKS
    OS_LIN --- AGENT
    OS_WIN --- AGENT
    UEFI_NET -.->|"Optional HTTPS (<=4s)"| SERVER
    CORE_REM -.->|"Optional mTLS WSS"| SERVER
    AGENT -.->|"Optional mTLS HTTPS"| SERVER
    CORE_ENG -.->|"Verified Downloads"| CATALOG
```

---

## 2. Repository & Component Dependency Graph

```mermaid
flowchart LR
    DOCS["docs (Specs & ADRs)"]
    RESEARCH["research (Primary Studies)"]
    SBM["sbm (Coordination & Roadmap)"]
    COMMON["sbm-common (Rust Types, Protocol, Schemas)"]

    UEFI["sbm-uefi (x86_64 & aarch64 EFI)"]
    CORE["sbm-core (Minimal Linux OS)"]
    AS["sbm-as (Apple Silicon Bootstrap)"]
    INSTALL["install (SBM Installer/Updater)"]
    SERVER["sbm-server (Self-Hosted Control Plane)"]
    CATALOG["sbm-catalog (OS Catalog & Recipes)"]
    AGENT["sbm-agent (Guest OS Daemon)"]
    LAB["sbm-lab (QEMU/OVMF/U-Boot Tests)"]
    WEB["smart-boot-menu.github.io (Website)"]

    RESEARCH --> DOCS
    DOCS --> SBM
    DOCS --> COMMON

    COMMON --> UEFI
    COMMON --> CORE
    COMMON --> AS
    COMMON --> INSTALL
    COMMON --> SERVER
    COMMON --> CATALOG
    COMMON --> AGENT
    COMMON --> LAB

    UEFI --> AS
    CORE --> AS
    UEFI --> INSTALL
    CORE --> INSTALL
    AS --> INSTALL
    CATALOG --> CORE
    CATALOG --> SERVER

    UEFI --> LAB
    CORE --> LAB
    SERVER --> LAB
    AGENT --> LAB
    INSTALL --> LAB
    SBM --> WEB
    DOCS --> WEB
```

---

## 3. Normal Boot Sequence (Fast Path — No Linux Initialized)

```mermaid
sequenceDiagram
    autonumber
    participant FW as UEFI Firmware
    participant UEFI as sbm-uefi
    participant ESP as ESP (\EFI\sbm)
    participant OS as Target OS (Windows/Linux/BSD)

    FW->>UEFI: LoadImage + StartImage (sbm-uefi.efi)
    UEFI->>UEFI: Init alloc, monotonic clock, GOP framebuffer
    UEFI->>ESP: Read local policy.json & last_attempt.json
    UEFI->>UEFI: Scan ESPs for Windows, BLS, UKI, EFI targets
    UEFI->>UEFI: Paint graphical menu (< 250 ms target)
    alt User selects target or countdown expires
        UEFI->>ESP: Write last_attempt.json (status=Started, failures+1)
        UEFI->>FW: LoadImage(target DevicePath) + StartImage
        FW->>OS: Transfer execution to OS bootloader
        Note over OS: Optional sbm-agent confirms boot success
    else Previous default exceeded max_failed_attempts
        UEFI->>UEFI: Auto-select fallback target or stay in menu
    end
```

---

## 4. Remote Boot Selection Sequence (Bounded & Offline-Safe)

```mermaid
sequenceDiagram
    autonumber
    participant OP as Operator (Web/Passkey)
    participant SRV as sbm-server
    participant UEFI as sbm-uefi
    participant ESP as ESP (\EFI\sbm)
    participant OS as Selected OS

    OP->>SRV: Select next boot target (scope=boot_select)
    SRV->>SRV: Sign CommandEnvelope (Ed25519, seq=N, TTL)
    Note over UEFI: Device powers on / reboots
    UEFI->>ESP: Check allow_remote_boot_select & pinned keys
    alt Enrolled & NIC Link Up
        UEFI->>SRV: HTTPS GET /v1/devices/{id}/boot-policy (timeout <= 4s)
        alt Valid Response Within Timeout
            SRV-->>UEFI: Signed CommandEnvelope (seq=N, target=X)
            UEFI->>UEFI: Verify Ed25519 sig, device_id, seq > last_seq, scope==boot_select
            UEFI->>ESP: Persist last_seq=N & mark one_shot consumed
            UEFI->>OS: Chainload target X
        else Timeout / Offline / Invalid Signature
            UEFI->>UEFI: Log diagnostic & fall back seamlessly to local policy
        end
    else Offline or User Pressed Local Key
        UEFI->>UEFI: Skip network immediately; follow local user/policy
    end
```

---

## 5. Recovery Environment Startup (`sbm-core`)

```mermaid
sequenceDiagram
    autonumber
    participant UEFI as sbm-uefi
    participant ESP as ESP (\EFI\sbm)
    participant UKI as sbm-core UKI (Slot A/B)
    participant INIT as systemd initrd
    participant CORE as sbm-core Userspace

    UEFI->>ESP: Read recovery/manifest.json (Slot A primary, Slot B backup)
    UEFI->>ESP: Write handoff/current.json + compute SHA-256
    UEFI->>UKI: Verify digest / Secure Boot -> LoadImage + StartImage
    UKI->>INIT: Boot Linux kernel + initramfs (cmdline: sbm.handoff_sha256)
    INIT->>INIT: Verify dm-verity Merkle tree on EROFS rootfs
    alt RAM >= 4 GiB (sbm.toram=auto)
        INIT->>INIT: Copy EROFS image to tmpfs/zram (frees boot media!)
    else Low RAM (< 4 GiB)
        INIT->>INIT: Mount EROFS read-only + zram overlayfs upper
    end
    INIT->>CORE: Switch root -> start systemd, labwc (+XWayland), sbm-cored
    CORE->>ESP: Verify handoff SHA-256 & mark recovery slot healthy
    CORE-->>CORE: Launch Iced Management UI (+ GParted & optional wayvnc)
```

---

## 6. OS Installation Workflow

```mermaid
sequenceDiagram
    autonumber
    participant USER as User / Remote Operator
    participant CORE as sbm-core Engine
    participant CAT as sbm-catalog (TUF Verified)
    participant DISK as Target Storage

    USER->>CORE: Choose OS & release from catalog
    CORE->>CAT: Verify TUF root/timestamp/snapshot/targets metadata
    CORE->>CORE: Filter compatible arch & platform (e.g. Apple vs x86)
    CORE->>DISK: Read-only probe -> build DiskIdentity & ProtectionClasses
    CORE->>CORE: Compile StorageOperationPlan + OsInstallPlan
    CORE-->>USER: Present dry-run diff & non-reversibility warning
    USER->>CORE: Explicit confirmation (+ Passkey step-up if remote)
    CORE->>CAT: Download OS artifact(s) (or load offline local media)
    CORE->>CORE: Verify exact SHA-256 against signed catalog manifest
    CORE->>DISK: Re-verify DiskIdentity & partition table hash
    CORE->>DISK: Apply partitions, format, extract/wimlib/bootc, install EFI entry
    CORE->>CORE: Register new BootTarget & offer reboot
```

---

## 7. Apple Silicon Boot Sequence (`sbm-as`)

```mermaid
flowchart TD
    BROM["Apple Boot ROM (Silicon Root of Trust)"] --> LLB["LLB (Verifies System Firmware & LocalPolicy)"]
    LLB --> IBOOT["iBoot2 / mBoot (Permissive Security via 1TR bputil)"]
    IBOOT -->|"macOS Volume Selected in Apple Picker"| MACOS["macOS (Signed System Volume)"]
    IBOOT -->|"SBM Stub Selected (Custom Boot Object)"| M1N1["m1n1 Stage 1/2 (Internal APFS Stub)"]
    M1N1 --> UBOOT["Asahi U-Boot (CONFIG_EFI_LOADER, GOP, NVMe, USB xHCI)"]
    UBOOT --> SBMUEFI["sbm-uefi (Standard aarch64-unknown-uefi Binary)"]
    SBMUEFI -->|"Fast Boot: Internal or External USB3 SSD"| ASAHIOS["Compatible ARM64 OS (linux-asahi 16 KiB kernel)"]
    SBMUEFI -->|"Management Mode"| SBMCORE["sbm-core-aarch64-apple (Light Fedora Asahi < 1 GB)"]
    SBMUEFI -.->|"Switch to macOS"| REBOOTMAC["Reboot to Apple Startup Target (No EFI Chainload)"]
```

---

## 8. Secure Software Update Workflow (TUF + A/B Rollback)

```mermaid
sequenceDiagram
    autonumber
    participant UPD as install / sbm-core Updater
    participant TUF as TUF Repository (Releases)
    participant SLOT as Inactive Slot (Slot B)
    participant UEFI as sbm-uefi

    UPD->>TUF: Fetch TUF timestamp -> snapshot -> targets (verify threshold sigs)
    UPD->>TUF: Download new sbm-core UKI + EROFS + sbm-uefi.efi
    UPD->>UPD: Verify SHA-256, SBAT generation, & signature bundle
    UPD->>SLOT: Write new image to inactive Slot B (Slot A untouched!)
    UPD->>UEFI: Set one-shot test boot to Slot B (max_failed_attempts=1)
    Note over UEFI: System Reboots
    UEFI->>SLOT: Boot Slot B (mark attempt Started)
    alt Slot B Boots & Self-Test Passes
        SLOT->>UEFI: Confirm healthy -> promote Slot B to active default
    else Boot Hangs / Verification Fails / Panic
        UEFI->>UEFI: Next reboot automatically rolls back to known-good Slot A!
    end
```

---

## 9. Remote Authorization Workflow (Scoped & Step-Up Verified)

```mermaid
flowchart TD
    OP["Operator Action in sbm-server Web UI"] --> AUTHN{"Authenticated via WebAuthn Passkey?"}
    AUTHN -- No --> DENY1["Reject (401 Unauthorized)"]
    AUTHN -- Yes --> SCOPE{"Required AuthScope?"}
    SCOPE -- "read_only / boot_select" --> RBAC1{"Role grants scope?"}
    SCOPE -- "os_install / storage_destructive" --> STEPUP{"Fresh Passkey Step-Up + Plan Hash Confirm?"}
    STEPUP -- No --> DENY2["Reject (403 Step-Up Required)"]
    STEPUP -- Yes --> RBAC2{"Role grants destructive scope?"}
    RBAC1 -- No --> DENY3["Reject (403 Forbidden)"]
    RBAC2 -- No --> DENY3
    RBAC1 -- Yes --> SIGN["Sign CommandEnvelope (Ed25519, DeviceID, Monotonic Seq, TTL)"]
    RBAC2 -- Yes --> SIGN
    SIGN --> AUDIT["Append Immutable Audit Log Entry"]
    AUDIT --> DEVICE{"Target Environment?"}
    DEVICE -- "sbm-uefi" --> UEFI_CHK{"Scope == boot_select?"}
    UEFI_CHK -- No --> REJECT_UEFI["sbm-uefi Hard-Rejects Non-BootSelect Scope"]
    UEFI_CHK -- Yes --> EXEC_UEFI["Verify Sig + Seq + TTL -> Apply Boot Policy"]
    DEVICE -- "sbm-core" --> CORE_CHK["Verify Sig + Seq + TTL + Local Device Policy -> Execute Plan"]
```

---

## 10. Storage Operation Lifecycle (Safety-First Execution)

```mermaid
stateDiagram-v2
    [*] --> ReadOnlyDiscovery
    ReadOnlyDiscovery --> PlanCompiled: Inspect DiskIdentity & ProtectionClasses
    PlanCompiled --> BlockedProtected: Touches ProtectedAppleSystem / ActiveESP without override
    BlockedProtected --> [*]
    PlanCompiled --> DryRunReview: Validated non-overlapping ops + table hash
    DryRunReview --> Cancelled: User declines
    Cancelled --> [*]
    DryRunReview --> Confirmed: Explicit confirmation + non-reversibility ack
    Confirmed --> PreFlightLockCheck: Re-verify live DiskIdentity & partition table SHA-256
    PreFlightLockCheck --> AbortedMismatch: Disk changed since planning!
    AbortedMismatch --> [*]
    PreFlightLockCheck --> Executing: Invoke audited CLI tools via execve (no shell)
    Executing --> VerifiedComplete: Sync + re-probe + audit log
    Executing --> FailedPartial: I/O error or power loss -> report exact step & recovery guidance
    VerifiedComplete --> [*]
    FailedPartial --> [*]
```

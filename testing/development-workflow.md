# Smart Boot Menu — Tiered Development Workflow & Quick-Start

**Status:** Active Engineering Standard (2026-11-26)  
**Related:** [ADR-008](../architecture-decisions/ADR-008-testing-virtualization-and-benchmark-methodology.md), [ADR-009](../architecture-decisions/ADR-009-dual-architecture-offline-preview-media.md), [ADR-010](../architecture-decisions/ADR-010-open-source-reuse-and-engineering-principles.md), [ADR-011](../architecture-decisions/ADR-011-native-preview-internal-install-and-media-creator.md), [`.github/AGENTS.md`](https://github.com/smart-boot-menu/.github/blob/main/AGENTS.md)

Smart Boot Menu separates ordinary software development from release engineering. Always validate changes at the **cheapest sufficient level** to keep feedback loops sub-second and minimize RAM, CPU, energy, and SSD wear on 16 GiB developer laptops.

---

## Quick-Start (`just`)

The workspace task runner lives canonically at [`sbm/justfile`](https://github.com/smart-boot-menu/sbm/blob/main/justfile) (and can be symlinked at the workspace root: `ln -sfn sbm/justfile justfile`).

```bash
# Inspect tools, host memory/VM reservations, and disk space (read-only)
just doctor
just status
just storage

# Level 0: Fast native host unit tests across core repos (< 10 s total)
just test                     # sbm-common, sbm-as, sbm-uefi, sbm-core, sbm-catalog, sbm-server
just test install             # uses cargo-nextest automatically when installed (46 s vs 85 s)
just check sbm-uefi           # fmt + clippy + unit tests

# Level 0: Host UI previews (MOCK / Tier 1 only — never release qualification)
just preview-uefi             # static sbm-gfx software render (~0.17 s)
just preview                  # interactive native preview (Track A, owned by Agent 18)

# Level 1: Native-architecture UEFI build + QEMU HVF/KVM smoke (~2–5 s)
just efi-build
just efi-smoke

# Level 2: Incremental Linux development
just core-zigbuild            # zero-VM macOS->Linux aarch64 build of sbm-cored (~0.5 s warm!)
just builder-start            # starts persistent ARM64 builder (reserves via sbm-lab resources)
just core-cargo sbm-cored     # warm incremental build inside builder (~5 s)
just core-build               # dev profile: native arch, incremental mkosi, uncompressed
just core-smoke               # boots disposable QCOW2 overlay
just builder-trim             # reclaims deleted guest blocks in QCOW2 via fstrim
just builder-stop

# Level 3 & 4: Explicit, heavy integration and release qualification (opt-in only)
just lab-smoke
just lab-integration path/to/image.raw
just release-core             # dual-arch compressed build (heavy)
just release-media            # assemble release media from lockfile
just release-repro            # reproducibility check (auto-prunes duplicate images)

# Maintenance (dry-run by default — never auto-deletes)
just clean-dry-run
```

---

## The Five Development & Validation Levels

| Level | Scope | Typical Latency | Guest RAM | Qualification Tier |
|---|---|---:|---:|---|
| **0 — Native Host** | Unit/property tests, synthetic disk/ESP fixtures, `sbm-plan`/`sbm-exec` regular-file simulation, mock UI previews | `0.1–10 s` (`46 s` for `install` via `nextest`) | `0 MiB` | Tier 1 (Simulated) |
| **1 — Minimal Native EFI** | Native-arch `sbm-uefi` build + QEMU HVF/KVM smoke (`xtask build/qemu native`) | `2–5 s` | `256–512 MiB` | Tier 2 (EFI smoke) |
| **2 — Incremental Linux** | `cargo-zigbuild` zero-VM binary cross-build (`sbm-cored`/`sbm-runtime`), persistent ARM64 builder, `SBM_BUILD_PROFILE=dev` uncompressed single-arch `mkosi --incremental=yes`, disposable `boot-smoke.sh` overlay | `0.5 s` (`zigbuild`) / `5 s` (in-VM) / `~2 min` (incremental image) | `0` or `4096 MiB` | Tier 2 (Dev) |
| **3 — Full-System Integration** | `sbm-lab test --tier integration` (UEFI -> Spec 03 handoff -> dm-verity EROFS -> `labwc` + Iced `sbm-ui` -> GParted -> reboot confirmation) | `~1 min` HVF / `~2.6 min` TCG | `2048–4096 MiB` | Tier 2 (Qualification) |
| **4 — Release Qualification** | `SBM_BUILD_PROFILE=release`, `zstd -19`, `check-reproducible.sh`, hosted Linux KVM CI, optional Tier 3 physical hardware | `~11+ min` | `4096 MiB` (or CI) | Tier 2 / Tier 3 |

### Non-Negotiable Qualification Rules

1. **Mocks are never release qualification.** Static renders (`just preview-uefi`) and interactive native previews (`just preview`, Track A / Agent 18) MUST display a visible `MOCK` / `SIMULATED (Tier 1)` indicator and never substitute for Tier 2 VM firmware, kernel, or block-device qualification.
2. **Native architecture by default.** Local builds default to the host architecture (`aarch64` on Apple Silicon, `x86_64` on x86-64 hosts) with compression disabled. Cross-architecture `x86_64` TCG emulation and dual-architecture images are explicit opt-in operations (see [ADR-009](../architecture-decisions/ADR-009-dual-architecture-offline-preview-media.md)).
3. **Global VM resource admission.** Always check `just status` (`sbm-lab resources status`) before starting a VM. Heavy and graphical VMs run sequentially within the 16 GiB host guest budget (`5120 MiB`).

---

## Cargo Build-Directory Isolation, Caching & Tooling Guidance

All recommendations below are backed by empirical measurements on the 16 GiB Apple Silicon M5 workstation (archived in `research/validation/evidence/sbm-dev-resource-audit-2026-11-26.md`):

### 1. Standard Cargo 1.100 `build.build-dir` Isolation (Recommended)

Cargo 1.100 stabilizes per-workspace build-directory templates. To keep intermediate Cargo build artifacts cleanly out of git working trees (without cross-worktree lock contention from an unhashed shared `CARGO_TARGET_DIR`, see `cargo#17312`), add to `~/.cargo/config.toml`:

```toml
[build]
build-dir = "{cargo-cache-home}/build/{workspace-path-hash}"
```

### 2. `cargo-nextest` (Recommended for `install` & `sbm-lab`)

- **`install` (61 tests across 3 heavy integration binaries):** `85.05 s` (`cargo test` baseline) -> **`46.33 s`** (`cargo nextest run`, **1.84× faster**, −56% sys CPU).
- **`sbm-lab` (52 tests):** `3.62 s` -> **`3.26 s`** (10% faster).
- `just test install` automatically invokes `cargo nextest run` when `cargo-nextest` is installed (override with `SBM_NO_NEXTEST=1`).

### 3. `cargo-zigbuild` (Recommended for Zero-VM `sbm-cored` / `sbm-runtime` Iteration)

- Cross-compiling `sbm-cored` for `aarch64-unknown-linux-gnu` directly on macOS via `just core-zigbuild`:
  - Cold build: **`9.14 s`** (262 MiB target dir, 0 MiB VM RAM).
  - Warm incremental rebuild: **`0.50 s`** (vs `~5.1 s` inside the 4 GiB Linux VM — **10× faster with zero VM memory**).
- Applies to pure-Rust Linux binaries (`sbm-cored`, `sbm-ctl`, `sbm-runtime`); `sbm-ui` Wayland/Linux GUI packaging and `mkosi` image assembly still run in Linux.

### 4. `sccache` (Opt-In Only — NOT Recommended Globally on Local Single-Worktree Hosts)

Benchmarked with `SCCACHE_ALLOW_INCREMENTAL=1` (`sccache 0.20.1`):
- **Same-repo fresh worktree (`sbm-server`):** `25.85 s` plain -> `19.64 s` warm `sccache` (**−24% wall time**, 85.5% hit rate, 328 MiB cache).
- **Cold miss (`sbm-server`):** `25.85 s` plain -> `32.89 s` (`+27%` slower).
- **Cross-repo (`sbm-core` after `sbm-server`):** `14.86 s` plain -> `19.15 s` (`+29%` slower, only 13.8% hit rate due to distinct crate features/versions + uncacheable `bin`/`proc-macro`/`incremental` units; cache grew to 519 MiB).
- **Guidance:** Do **not** set `RUSTC_WRAPPER=sccache` globally for single-worktree local development. Enable it selectively in CI or when frequently creating fresh `git worktree` checkouts of `sbm-server`.

---

## Safe Storage Management & Retention

- Always inspect before deleting: `just storage` and `just clean-dry-run` are strictly read-only.
- Use `just builder-trim` (`fstrim` with `discard=unmap`) to shrink the builder QCOW2 after pruning guest files (reclaimed 3.63 GiB in baseline audit).
- Use APFS copy-on-write clones (`cp -c`) when staging identical `.raw` inputs across repositories on macOS.
- **Protected paths (never auto-cleaned):** `sbm-core/build/evidence/`, `install/build/out/release/`, `sbm-lab/var/runs/int-*`, `sbm-catalog/cache/`, backing base images, and `~/.cargo/registry/`.

---

## Linux Builder Backend Status (QEMU HVF vs Lima + `krunkit`)

Benchmarked head-to-head on 2026-11-26 (`Fedora 45 aarch64`, `4096 MiB`, `2 vCPUs`; raw data in `research/validation/evidence/sbm-dev-resource-audit-2026-11-26.md` §8.2):
- **Lima `2.3.0` + `krunkit 1.3.2` (`libkrun 1.19.8`):** `1.85×` faster warm boot (`8.83 s` vs `16.32 s`), zero-copy `virtiofs` source mount, identical `0.31 s` warm incremental `sbm-cored` build, working `fstrim` and `unshare -r`. However, `krunkit 1.3.2` does **not** attach a `virtio-balloon` device yet (`krunkit#121`), so freed guest RAM was not reclaimed on macOS (`1151 -> 2010 -> 2018 MiB`; post-build `3782 -> 3756 MiB`).
- **Decision:** Keep the hardened QEMU HVF builder (`sbm-core/scripts/start-builder.sh` with `discard=unmap`, `fstrim`, `SBM_BUILD_PROFILE=dev`, and native `sbm-lab resources` admission) as default for `mkosi` image builds, use host `just core-zigbuild` (`0.50 s`, `0 MiB` VM RAM) for pure-Rust daemon iteration, keep QEMU for all firmware testing, and re-evaluate Lima + `krunkit` once `krunkit#121` wires `libkrun` balloon reclaim.

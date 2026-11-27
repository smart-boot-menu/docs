# ADR-010: Open-Source Reuse & Engineering Principles

- **Status:** Accepted (Foundational, Organization-Wide)
- **Date:** 2026-11-26
- **Scope:** All `smart-boot-menu` repositories, contributors, and autonomous agents
- **Canonical Agent Policy:** [`.github/AGENTS.md`](https://github.com/smart-boot-menu/.github/blob/main/AGENTS.md) & [`.github/agent-reference.md`](https://github.com/smart-boot-menu/.github/blob/main/agent-reference.md)
- **Related ADRs:** ADR-002, ADR-003, ADR-005, ADR-006, ADR-007, ADR-008

## Context

Smart Boot Menu spans pre-OS UEFI software, minimal Linux recovery environments, Apple Silicon bootstrap integration, storage/OS provisioning, optional remote management, declarative OS catalogs, installers, guest agents, and virtual hardware testing. Across these domains, mature open-source software already solves many hard problems.

Earlier decisions (such as ADR-003 reusing `mkosi` and Fedora Minimal, ADR-005 reusing `m1n1` and `AsahiLinux/u-boot`, ADR-006 wrapping mature storage and filesystem tools rather than rewriting them in Rust, and ADR-008 reusing QEMU, EDK II, U-Boot, and `swtpm`) applied reuse pragmatically in specific domains. However, without a permanent organization-wide policy, contributors and autonomous agents can default to writing custom implementations, rewriting working non-Rust software in Rust for language preference alone, or retaining inferior custom code due to sunk-cost bias.

Our objective is to advance the state of the art, not duplicate it.

## Guiding Principle

**Do not recreate inferior versions of existing software.**

**Write less software. Reuse the best available implementations. Make meaningful contributions where existing solutions fall short.**

We write original software only when we can provide meaningful, demonstrable improvements over existing solutions.

## Decision

Adopt the following eight permanent, organization-wide engineering principles across all Smart Boot Menu repositories:

### 1. Research Before Implementation

Before designing or implementing a new component, investigate existing open-source projects that solve the same or substantially similar problems.

- Search beyond familiar programming languages, ecosystems, and repositories.
- Examine actual implementations, documentation, limitations, benchmarks, licensing, and maintenance status.
- Do not assume that a solution does not exist simply because you are unfamiliar with it.
- Record reusable ecosystem findings in `smart-boot-menu/research/`.

### 2. Order of Preference

Prefer, in order:

1. **Reuse existing software directly.**
2. **Configure and compose existing software.**
3. **Write thin integration layers.**
4. **Contribute improvements upstream.**
5. **Maintain a fork when justified.**
6. **Develop an original implementation when it offers meaningful advantages.**

This ordering is a default engineering preference, not an absolute constraint.

### 3. Justify Original Implementations

Writing an original implementation must provide a demonstrable advantage in one or more areas:

- Functionality
- Performance
- Energy efficiency
- Memory consumption
- Storage consumption
- Security
- Reliability
- Portability
- Maintainability
- User experience
- Developer experience
- Architectural simplicity

Evaluate improvements against the **strongest relevant existing alternatives**, not artificially weak baselines. Consider **total cost of ownership (TCO)**, including dependencies, integration, ongoing maintenance, security updates, and future development.

### 4. Language Is Not Justification

Rust is our preferred language for original systems software.

However, rewriting an existing C, C++, Python, or other implementation in Rust is **not** inherently an improvement. Language choice must produce tangible benefits (for example, verified security, reliability, `#![no_std]` UEFI portability, or resource-efficiency gains that outweigh TCO) to justify a rewrite.

### 5. Avoid Sunk-Cost Reasoning

Apply these principles retrospectively to existing Smart Boot Menu code.

If an existing open-source project provides a superior solution, be willing to replace, simplify, or remove our own implementation. Previously invested engineering effort is not sufficient justification for retaining inferior software.

### 6. Contribute Upstream

When an improvement is generally useful and not specific to Smart Boot Menu, prefer contributing it to the upstream project. Avoid unnecessary fragmentation of the open-source ecosystem.

### 7. Keep Research Proportional

Do not turn every minor function or bug fix into an extensive research project.

The depth of research must be proportional to the complexity, novelty, risk, and long-term maintenance cost of the proposed implementation:

- **Routine changes** (bug fixes, small internal helpers, test additions, incremental refinements within an already-justified architecture) do not require a formal build-versus-reuse study.
- **Significant additions or changes** (new subsystems, major dependencies, architectural changes, and replacements of established software) require an explicit build-versus-reuse justification recorded in `research/`, an ADR in `docs/architecture-decisions/`, or the component's `IMPLEMENTATION_PLAN.md` before major implementation begins.

### 8. Evidence Over Assumptions

Distinguish measured results from theoretical advantages.

- Do not claim that an SBM implementation is faster, lighter, safer, or better without appropriate evidence.
- When practical, establish an empirical baseline against existing alternatives before implementing a replacement, and archive raw evidence in `smart-boot-menu/research/validation/evidence/` following `docs/testing/benchmarks.md`.

## Preservation of Security, Safety, Licensing & Offline-First Invariants

Reuse never overrides Smart Boot Menu's non-negotiable invariants:

- **Licensing:** Original SBM code and documentation remain dual-licensed `MIT OR Apache-2.0`. Copyleft CLI tools (`GPL-2.0`/`GPL-3.0`) must remain isolated as separate subprocesses invoked via direct `execve` / `Command::args` (never shell interpolation), and `GPL`/`AGPL` crates must not be statically linked into permissive SBM Rust crates.
- **Security & Safety:** Reused components must still satisfy `docs/threat-model/README.md` (offline-first local operation, zero arbitrary RCE in `sbm-uefi`, safe read-only disk discovery, stable `DiskIdentity` guardrails, no committed secrets, and workstation safety).
- **Release & Quality Gates:** Reused and original components alike remain subject to deterministic verification, reproducible builds, and documented qualification gates.

## Operational Enforcement

- **Central Agent Documentation (0SFS Pattern):** All universal rules, operational checklists, and repository-specific guardrails live centrally in `.github/AGENTS.md` and `.github/agent-reference.md` rather than duplicated per repository.
- **Minimal Repository Pointer Stubs:** Each individual repository's `AGENTS.md` is a minimal 5-line pointer referencing `smart-boot-menu/.github/AGENTS.md` (both the GitHub URL for independently launched single-repo agents and the local `../.github/AGENTS.md` path).
- **Orchestration & Templates:** Agent launch prompts (`sbm/agents/`), pull request checklists (`.github/PULL_REQUEST_TEMPLATE.md`), and RFC templates (`.github/ISSUE_TEMPLATE/rfc_interface.yml`) require proportional build-versus-reuse evaluation.
- **Single Source of Truth:** Do not copy agent policies or repository rulebooks into individual repositories; maintain them centrally in `.github/AGENTS.md`, `.github/agent-reference.md`, and this ADR.

## Consequences

- **Positive:** Smaller custom codebase to audit and maintain; faster progress by standing on battle-tested upstream ecosystems; fewer language-motivated rewrites; clearer, evidence-backed architecture decisions; healthier upstream contributions.
- **Trade-offs:** Requires disciplined upfront evaluation for major components, careful license and supply-chain vetting of reused projects, and engineering humility to retire custom code when superior open-source alternatives exist.

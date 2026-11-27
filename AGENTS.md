# AGENTS.md — `docs`

## Canonical Organization Policy (Read First)

Before designing or implementing changes, read the canonical organization-wide engineering rules in [`smart-boot-menu/.github/AGENTS.md`](https://github.com/smart-boot-menu/.github/blob/main/AGENTS.md) (local workspace: `../.github/AGENTS.md`), the operational checklists in [`.github/agent-reference.md`](https://github.com/smart-boot-menu/.github/blob/main/agent-reference.md) (`../.github/agent-reference.md`), and the **Open-Source Reuse & Engineering Principles** in [`ADR-010`](architecture-decisions/ADR-010-open-source-reuse-and-engineering-principles.md) ([GitHub](https://github.com/smart-boot-menu/docs/blob/main/architecture-decisions/ADR-010-open-source-reuse-and-engineering-principles.md)).

> **Standalone / Single-Repo Agents:** If you were launched inside this repository alone and `../.github/AGENTS.md` is not present locally, fetch and follow the GitHub URLs above before proceeding. Do not recreate inferior versions of existing software; apply proportional build-versus-reuse evaluation for new components, major dependencies, architectural changes, or replacements.

## Repository Role

`smart-boot-menu/docs` is the **canonical owner of all cross-repository technical specifications, Architecture Decision Records (ADRs), threat models, hardware profiles, diagrams, and testing standards**.

## Ownership

- `architecture-decisions/` — Numbered ADRs documenting significant technology choices, alternatives, trade-offs, and open experiments.
- `specifications/` — Versioned wire, file, and schema specifications implemented canonically in `sbm-common`.
- `threat-model/` — System-wide threat model and security invariants.
- `hardware/` — Target hardware characterization and platform caveats.
- `testing/` — Benchmark methodology, engineering targets, and integration test matrices.
- `diagrams/` — Canonical Mermaid architecture and sequence diagrams.
- `proposals/` — RFCs and open design proposals prior to ADR ratification.
- `validation/` — Validation layout and acceptance records.

## Boundaries (Do NOT Duplicate)

- Do NOT place raw ecosystem surveys or raw benchmark dumps here — those belong in `smart-boot-menu/research`.
- Do NOT write Rust code or JSON schemas here — canonical Rust types and generated JSON schemas belong in `smart-boot-menu/sbm-common`.
- When updating any specification under `specifications/`, you MUST open a coordinated PR in `sbm-common` updating the Rust types, JSON schemas, and test vectors.

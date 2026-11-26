# AGENTS.md — `docs`

Follow the universal Smart Boot Menu agent rules in [`smart-boot-menu/.github/AGENTS.md`](https://github.com/smart-boot-menu/.github/blob/main/AGENTS.md) (locally `../.github/AGENTS.md`).

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

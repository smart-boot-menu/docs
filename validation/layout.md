# Validation & Knowledge Layout

Modeled on `0SFS/docs/validation/layout.md`, Smart Boot Menu enforces a clean separation between normative specifications, primary-source research, retained empirical evidence, and runnable tests:

| Kind of Artifact | Canonical Repository & Path |
|---|---|
| Normative interface specs & versioning rules | `smart-boot-menu/docs/specifications/` |
| Architecture Decision Records (ADRs) | `smart-boot-menu/docs/architecture-decisions/` |
| Threat model & safety invariants | `smart-boot-menu/docs/threat-model/` |
| Mermaid architecture & sequence diagrams | `smart-boot-menu/docs/diagrams/` |
| Benchmark methodology & engineering targets | `smart-boot-menu/docs/testing/benchmarks.md` |
| Primary-source ecosystem & hardware research | `smart-boot-menu/research/{ecosystem,hardware,architecture}/` |
| Retained empirical benchmark & probe evidence | `smart-boot-menu/research/validation/evidence/` |
| Executable Rust types, JSON schemas, test vectors | `smart-boot-menu/sbm-common/{crates,schemas,test-vectors}/` |
| Runnable QEMU/OVMF/AAVMF/U-Boot integration & fault tests | `smart-boot-menu/sbm-lab/` |
| Scratch / temporary artifacts | Local `/tmp/` or untracked scratch dirs — NEVER committed to git and NEVER placed in `~/.gemini/` |

# Noticer

Noticer is a privacy-conscious Rust player tracking and intel tool for a private Discord group.

This repository currently contains the product plan, audit findings, architecture/file-structure proposal, and Rob action items.

## Files

- [`docs/noticer-plan.md`](docs/noticer-plan.md) — revised integrated plan.
- [`docs/audit-findings.md`](docs/audit-findings.md) — harsh audit findings and gaps.
- [`docs/rob-todos.md`](docs/rob-todos.md) — owner/operator setup checklist for Rob.
- [`docs/architecture.md`](docs/architecture.md) — proposed architecture pattern and repo layout.

## Guardrails

Noticer must stay limited to in-game/public platform identity, private/team-only intel, and user-confirmed evidence. It must not collect or publish doxxing material, private addresses, private contact details, IP addresses, or server-admin-only identifiers unless Rob actually operates the server and has a legal/ToS-compliant reason to process that data.

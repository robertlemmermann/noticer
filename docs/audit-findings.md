# Noticer Audit Findings

## Verdict

The original plan is already stronger than most hobby-tool plans on security, auditability, backups, and Discord delivery. The remaining gaps are mostly product/UX reality, data quality, privacy/legal minimization, operational failure modes, and architecture specificity.

The most important correction: the tool should not be framed as tracking “real identity.” It should track in-game/platform identity only. Anything that looks like doxxing, harassment support, or public shaming creates disproportionate legal and platform-risk exposure.

## Critical gaps found

### 1. Dossier consumer experience is under-specified

Problem: The plan describes what the dossier shows, but not how a Discord user decides whether to trust it in 5 seconds.

Fixes added:

- Every Discord embed needs a compact “reason summary,” not just badges.
- Every negative label needs evidence/provenance and a confidence level.
- Staleness must be visible: `last_synced`, `last_verified_by_human`, and stale warning state.
- Embeds must avoid wall-of-text. Use one primary claim, 3 supporting facts, and buttons for detail.
- Use neutral phrasing for risky labels: `suspected cheater` with evidence, not naked `cheater`.
- Add “wrong person?” / “report bad intel” CTA to private pages for correction workflow.

### 2. Data trust and source provenance are not first-class enough

Problem: BattleMetrics coplay, Steam public fields, manual notes, and group assignments have wildly different reliability. The original plan stores them together but does not force the UI to show source quality.

Fixes added:

- Add `evidence`, `source_kind`, `confidence`, and `verified_at` fields to claims.
- Relationship edges decay or become stale unless recently observed or manually confirmed.
- Coplay suggestions require multiple sessions, population thresholding, duration thresholding, and server diversity.
- Manual claims require a source note or “low confidence” default.
- Every field that could harm a reputation must show provenance before it is shareable.

### 3. “Track indefinitely” conflicts with privacy, storage, and correctness

Problem: Indefinite tracking is attractive for a Rust intel tool, but it creates stale intelligence, storage growth, and erasure conflict.

Fixes added:

- Keep objective history, but show “stale unless re-verified” for subjective claims.
- Add finite retention classes: hot normalized data, cold snapshots, immutable backup retention, and purge/tombstone behavior.
- Store only diffs for unchanged API payloads.
- Add erasure workflow: live purge + tombstone + backup expiry disclosure.

### 4. User management UX is too admin-centric and not enough “operator cockpit”

Problem: Admin features exist, but the plan lacks a single place that tells Rob whether the system is healthy, safe, and current.

Fixes added:

- Add Owner Cockpit: ingestion health, stale data, Discord failures, backup status, disk growth, API quota, unresolved security alerts, contributor activity spikes.
- Add setup wizard for first-run config.
- Add “safe mode” that disables public/unlisted sharing and outbound alerts during incidents.
- Add recovery drills and explicit RPO/RTO.

### 5. Contributor UX needs anti-footgun design

Problem: Autosave, bulk edits, and quick capture are fast but dangerous with multiple editors.

Fixes added:

- Pending/failed/saved states must be explicit on mobile.
- Bulk operations require preview, dry-run counts, and undo/revert path.
- Conflicting edits must be rejected at field level with a readable conflict UI.
- Add “draft note” behavior when network is weak.
- Add per-contributor trust level and moderation queue for new contributors.

### 6. Discord bot has hidden UX traps

Problem: Discord interactions have strict timing, limited embed space, permission failures, and DM delivery issues.

Fixes added:

- Every command defers immediately unless guaranteed cache-only.
- Autocomplete must be cache-only and capped.
- Channel permissions are checked at config time and periodically.
- Bot posts status warnings if `Embed Links`, `Send Messages`, or DM delivery fails.
- Alerts default to digests; instant alerts require explicit per-subscription opt-in.
- Every outbound message sets `allowed_mentions: { parse: [] }`.

### 7. Security plan is good but missing productized controls

Problem: The plan correctly names XSS/SSRF/CSRF/MFA/logging, but it needs implementation acceptance criteria.

Fixes added:

- Threat model document must exist before implementation.
- Central render-safe string type or formatting helper used by web, bot, and OG renderer.
- SSRF-safe URL parser and media proxy must be tested.
- Secrets rotation runbook per secret.
- RLS or explicit workspace scoping tests before multi-tenant use.
- Permission matrix tests for every role/action.
- Audit-log integrity verification job must alert, not merely compute.

### 8. Architecture pattern was too vague

Problem: “FastAPI + Next.js + worker + bot” is a deployment shape, not an architecture pattern.

Fixes added:

- Use a modular-monolith backend with hexagonal ports/adapters.
- Use an outbox/event pipeline for ingestion → notifications.
- Put all write operations behind command handlers with audit logging and authorization middleware.
- Separate objective domain data from workspace-scoped subjective intel.
- Define repo structure and boundaries.

### 9. Legal/ToS needs sharper product constraints

Problem: The original plan acknowledges privacy/ToS, but the product can still drift into a public shame database.

Fixes added:

- No indexable public dossiers.
- No real-world identity fields by default.
- No private contact info, addresses, IPs, or BEGUIDs unless server-operated and compliant.
- Shareable pages should be private by default for risky labels.
- Add correction/erasure workflow and anti-harassment policy.
- Add “publishability gate” that blocks sharing claims with insufficient evidence.

### 10. Testing plan needs abuse and data-quality scenarios

Problem: Normal unit/integration/e2e tests are not enough for this product.

Fixes added:

- Golden snapshot replay for API payloads.
- Fuzzing for hostile names, markdown, URLs, and HTML meta rendering.
- Permission-matrix tests.
- Bulk revert/unmerge restore tests.
- Notification dedupe/cooldown tests.
- Recovery test: restore backup into staging and verify hash chain.

## Highest-priority changes to make before any code

1. Rename/position the product as private in-game intel, not real-world tracking.
2. Add source provenance/confidence to every sensitive claim.
3. Define first-run owner cockpit and setup wizard.
4. Lock down dossier sharing before implementing it.
5. Choose architecture: modular monolith + outbox, not ad hoc services.
6. Write Rob’s external setup tasks separately so implementation is not blocked by missing accounts/tokens/domains.

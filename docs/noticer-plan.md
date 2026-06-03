# Noticer — Rust Player Intel and Private Dossier System

Status: revised audit-integrated plan
Scope: private/team-only Rust player intelligence using BattleMetrics, Steam Web API, and manually entered in-game intel
Default posture: private, evidence-based, reversible, auditable

---

## 1. Product definition

Noticer tracks Rust players over time for a private group: names, Steam/BattleMetrics identity, server history, associations, manually recorded encounters, and watch/alert preferences.

The product is **not** a public shaming database, doxxing tool, or real-world identity tracker. It must stay focused on in-game/platform identity and private operational intel.

### 1.1 Primary users

1. **Rob / owner** — installs, configures, audits, manages contributors, resolves incidents.
2. **Trusted editors** — add player notes, tags, sightings, and relationship confirmations.
3. **Viewers / teammates** — consume dossiers and Discord alerts; should understand the gist in seconds.
4. **Discord admins** — configure channels, bot permissions, role mapping, and alert delivery.
5. **Future multi-tenant owners** — not supported on day one, but schema must not block it.

### 1.2 North-star UX

The system is only useful if people can add and consume intel during actual gameplay.

Design targets:

- Add a player from a Steam/BattleMetrics URL in one paste.
- Record common intel in under 5 seconds on mobile.
- Consume a Discord dossier card in under 5 seconds.
- Show freshness, source, confidence, and evidence for every risky claim.
- Never make admins use SQL to fix bad data.
- Make system health obvious to Rob from one screen.

---

## 2. Non-negotiable guardrails

### 2.1 Privacy and legal guardrails

- Track **in-game/platform identity**, not real-world identity.
- Do not store private home addresses, phone numbers, private social accounts, employer details, family details, or other doxxing material.
- Do not publish Steam “real name” fields into Discord embeds or share pages by default; treat them as optional platform metadata only if public and relevant.
- Do not store or publish IPs/BattlEye GUIDs unless Rob operates the server, has lawful/ToS-compliant access, and explicitly enables a separate server-admin mode.
- No indexable public dossier tier. Only `private` and `unlisted`; default to `private` for profiles containing negative labels or sensitive notes.
- Every external share must pass a **shareability gate**: no unverified accusations, no private notes, no doxxing fields, and no claims without source/confidence.
- Provide a correction/erasure workflow: live purge/tombstone where appropriate, and disclose backup expiry for immutable backups.

### 2.2 Anti-harassment posture

Noticer may support private team awareness. It must not support harassment campaigns.

Product constraints:

- No mass-DM or public callout features.
- No “brigade,” “target this person,” or harassment workflow language.
- Risky labels use careful phrasing: `suspected cheater`, `reported offline raider`, `hostile`, `known ally`, etc.
- Negative labels require evidence/provenance before sharing.
- Owner can disable all sharing/alerts with **safe mode**.

### 2.3 Security baseline

- Treat Steam names, avatars, aliases, profile fields, BattleMetrics names, manual notes, media captions, and pasted URLs as hostile input.
- Centralize output encoding for web, Discord, and Open Graph rendering.
- No server-side fetching of user-influenced URLs unless parsed through a strict allowlist and private-IP redirect block.
- All writes are authorized, audited, and reversible.
- Owner/admin use Discord OAuth plus app-side TOTP MFA.
- Secrets are never committed and have rotation runbooks.
- Backups are encrypted before upload.

---

## 3. Data sources

### 3.1 BattleMetrics

Use BattleMetrics for:

- player search and lookup
- player profile data
- server history/session data where available
- coplay signals
- server metadata
- optional BattleMetrics flags/notes if API access confirms value

Implementation rules:

- Use authenticated requests.
- Implement token-bucket rate limiting and backoff.
- Build a Phase 0 spike to confirm exactly what the Premium token returns through the API, not just the web UI.
- Do not hard-depend on BattleMetrics flags/notes. Mirror Noticer-owned flags/notes in the local DB.
- Treat coplay as a suggestion source, not ground truth.

### 3.2 Steam Web API

Use Steam for:

- vanity URL resolution
- SteamID64 canonical identity
- current persona name/avatar/profile URL
- public country/profile state when available
- VAC/game/economy ban state
- friend list if public
- Rust playtime if public

Implementation rules:

- Batch calls where the API allows batching.
- Snapshot persona names on every poll to build local alias history.
- Do not scrape community alias endpoints as core behavior. If used at all, make it optional/manual import after ToS review.
- Treat all public Steam text as hostile input.

### 3.3 Manual intel

Manual intel includes:

- allegiance
- threat level
- group/team assignment
- relationship confirmation
- base location on a Rust server/wipe
- playstyle and build-style tags
- encounter notes
- media/evidence links
- corrections and disputes

Rules:

- Manual intel is workspace-scoped.
- Manual claims must store actor, timestamp, confidence, and source/evidence note.
- New contributors default to lower trust; their high-impact claims can require review before appearing in shared dossiers.

---

## 4. Data trust model

This was the biggest missing product layer in the original plan. Noticer must help users distinguish fact, inference, and opinion.

### 4.1 Claim model

Sensitive or subjective facts should be modeled as claims, not naked fields, when possible.

Examples:

- “enemy”
- “team member of X”
- “offline raider”
- “cheater suspect”
- “base near Launch”
- “known alt of Y”

Claim fields:

```txt
claim
  id
  workspace_id
  subject_type enum(player|group|server|relationship)
  subject_id
  claim_type text
  value jsonb
  confidence enum(low|medium|high|confirmed)
  source_kind enum(manual|battlemetrics|steam|discord|derived|import)
  evidence_summary text nullable
  evidence_url nullable
  created_by
  verified_by nullable
  verified_at nullable
  stale_at nullable
  deleted_at nullable
```

### 4.2 Provenance display

Any UI that shows a sensitive claim must also be able to show:

- who added it
- when it was added
- source type
- confidence
- last verified date
- evidence summary
- whether it is stale

Discord embeds show abbreviated provenance. Full pages show complete provenance.

### 4.3 Staleness

Subjective intel goes stale. Objective history does not disappear, but should be framed correctly.

Defaults:

- unconfirmed coplay suggestion stale after 14 days
- base location stale at wipe boundary or 14 days, whichever comes first
- threat notes stale after 90 days unless verified
- group membership stale after 30 days unless observed/confirmed
- bans never stale, but sync status can be stale

Stale intel remains visible in detail pages but is visually de-emphasized and excluded from short embeds unless manually pinned.

---

## 5. Core data model

### 5.1 Global objective facts

```txt
player
  id uuid pk
  steam_id64 text unique nullable
  bm_player_id text unique nullable
  current_name text
  avatar_url text nullable
  country text nullable
  profile_url text nullable
  profile_visibility text nullable
  vac_banned bool nullable
  game_ban_count int nullable
  economy_ban text nullable
  days_since_last_ban int nullable
  rust_hours_total int nullable
  rust_hours_recent int nullable
  watch_status enum(untracked|tracked|priority|dormant)
  sync_status enum(ok|partial|failed|stale|unknown)
  last_steam_synced_at timestamptz nullable
  last_bm_synced_at timestamptz nullable
  created_at timestamptz
  updated_at timestamptz
```

```txt
alias
  id uuid pk
  player_id fk
  name text
  source enum(steam|battlemetrics|manual|import)
  first_seen timestamptz
  last_seen timestamptz
  confidence enum(low|medium|high|confirmed)
```

```txt
identifier
  id uuid pk
  player_id fk
  type enum(steamID|battlemetricsID|vanity)
  value text
  source enum(steam|battlemetrics|manual)
  first_seen timestamptz
  last_seen timestamptz nullable
```

Do not include IP/BEGUID in the default schema. If server-admin mode is ever enabled, add them in a separate restricted table with explicit retention and access controls.

### 5.2 Workspace-scoped subjective intel

```txt
workspace
  id uuid pk
  name text
  owner_user_id fk app_user
  mode enum(single_tenant|multi_tenant_ready)
  created_at timestamptz
```

```txt
player_workspace_profile
  id uuid pk
  workspace_id fk
  player_id fk
  allegiance enum(enemy|ally|neutral|peaceful|unknown)
  threat_level int check 0 <= threat_level <= 5
  global_notes text nullable
  last_human_verified_at timestamptz nullable
  created_at timestamptz
  updated_at timestamptz
  UNIQUE(workspace_id, player_id)
```

```txt
server
  id uuid pk
  bm_server_id text unique
  name text
  region text nullable
  game text default 'rust'
  last_seen_rank int nullable
  last_synced_at timestamptz nullable
```

```txt
player_server
  id uuid pk
  player_id fk
  server_id fk
  first_seen_on_server timestamptz nullable
  last_seen_on_server timestamptz nullable
  time_played_seconds bigint nullable
  deleted_at timestamptz nullable
  UNIQUE(player_id, server_id) WHERE deleted_at IS NULL
```

```txt
player_server_intel
  id uuid pk
  workspace_id fk
  player_id fk
  server_id fk
  server_allegiance enum(enemy|ally|neutral|peaceful|unknown) nullable
  base_location text nullable
  server_notes text nullable
  reputation_local text nullable
  last_verified_at timestamptz nullable
  added_by fk app_user
  deleted_at timestamptz nullable
  UNIQUE(workspace_id, player_id, server_id) WHERE deleted_at IS NULL
```

### 5.3 Relationship graph

```txt
relationship
  id uuid pk
  workspace_id nullable  # null only for objective imported coplay; manual edges require workspace_id
  player_id fk
  other_player_id fk
  kind enum(team_member|ally|associate|enemy|coplay|alt_of|steam_friend)
  source enum(manual|battlemetrics|steam_friend|derived)
  strength int nullable
  confidence enum(low|medium|high|confirmed)
  server_id nullable
  confirmed bool default false
  evidence_summary text nullable
  first_seen timestamptz nullable
  last_seen timestamptz nullable
  stale_at timestamptz nullable
  notes text nullable
  deleted_at timestamptz nullable
  CHECK (player_id <> other_player_id)
```

Coplay rules:

- Coplay is auto-created as unconfirmed.
- Coplay is suggested only when it clears a noise floor.
- Suggestions must be dismissible, and dismissal should be remembered.
- Promotion to team/ally/associate/enemy requires explicit human confirmation.
- Relationship cards show the reason: “3 overlaps across 2 servers, 4h total, last seen 2026-06-01.”

### 5.4 Groups

```txt
group
  id uuid pk
  workspace_id fk
  name text
  color text nullable
  tag text nullable
  allegiance enum(enemy|ally|neutral|peaceful|unknown)
  confidence enum(low|medium|high|confirmed)
  notes text nullable
  deleted_at timestamptz nullable
```

```txt
group_member
  id uuid pk
  workspace_id fk
  group_id fk
  player_id fk
  role text nullable
  confidence enum(low|medium|high|confirmed)
  since timestamptz nullable
  stale_at timestamptz nullable
  added_by fk app_user
  deleted_at timestamptz nullable
```

### 5.5 Tags

```txt
tag
  id uuid pk
  workspace_id fk
  dimension enum(build_style|monument|playstyle|custom)
  label text
  color text nullable
  icon text nullable
  is_seed bool
  created_by fk app_user nullable
  created_at timestamptz
  deleted_at timestamptz nullable
  deleted_by fk app_user nullable
  UNIQUE(workspace_id, dimension, lower(label)) WHERE deleted_at IS NULL
```

```txt
player_tag
  id uuid pk
  workspace_id fk
  player_id fk
  tag_id fk
  server_id nullable
  confidence enum(low|medium|high|confirmed)
  evidence_summary text nullable
  added_by fk app_user
  added_at timestamptz
  deleted_at timestamptz nullable
```

### 5.6 Events and media

```txt
event
  id uuid pk
  workspace_id fk
  player_id fk nullable
  group_id fk nullable
  server_id fk nullable
  occurred_at timestamptz
  kind text check kind in ('raid','raided_by','betrayal','trade','encounter','sighting','note','correction','dispute')
  title text
  body text
  confidence enum(low|medium|high|confirmed)
  created_by fk app_user
  created_at timestamptz
  deleted_at timestamptz nullable
```

```txt
media
  id uuid pk
  workspace_id fk
  player_id fk nullable
  event_id fk nullable
  url text
  kind enum(image|clip|link)
  caption text nullable
  safety_status enum(pending|approved|blocked)
  added_by fk app_user
  added_at timestamptz
  deleted_at timestamptz nullable
```

Media rules:

- External media links are not blindly fetched.
- Render through proxy/cache only after allowlist, size/type checks, and private-IP redirect blocking.
- Media can be marked private and excluded from unlisted dossiers.

### 5.7 Dossiers

```txt
dossier
  id uuid pk
  workspace_id fk
  player_id fk
  slug text unique
  visibility enum(private|unlisted)
  risk_level enum(low|medium|high)
  created_by fk app_user
  created_at timestamptz
  revoked_at timestamptz nullable
  expires_at timestamptz nullable
```

```txt
dossier_access_log
  id uuid pk
  dossier_id fk
  accessed_at timestamptz
  requester_hash text nullable
  user_agent_hash text nullable
  result enum(allowed|revoked|auth_required|rate_limited|blocked)
```

Dossier access rules:

- `private` requires auth before meta tags or body render.
- `unlisted` uses long random slugs and is noindex.
- Revocation blocks future fetches but cannot recall cached previews.
- Risky labels are omitted or downgraded unless shareability gate passes.
- Dossier pages show source/confidence and “last verified.”

### 5.8 Audit and events

```txt
audit_log
  id uuid pk
  workspace_id nullable
  actor_user_id fk app_user nullable
  action text
  entity_type text
  entity_id uuid
  field text nullable
  before jsonb nullable
  after jsonb nullable
  sensitive bool default false
  prev_hash text nullable
  row_hash text not null
  at timestamptz
  request_meta jsonb nullable
```

```txt
outbox_event
  id uuid pk
  workspace_id nullable
  event_type text
  entity_type text
  entity_id uuid
  payload jsonb
  created_at timestamptz
  processed_at timestamptz nullable
  failed_at timestamptz nullable
  failure_reason text nullable
```

Audit rules:

- App DB role can insert/select audit rows but not update/delete them.
- Hash-chain verification runs on a schedule and alerts owner on mismatch.
- Off-box append-only log shipping mirrors sensitive audit events.
- Revert operations create new audit entries; they do not edit history.

### 5.9 Snapshots and storage lifecycle

```txt
snapshot
  id uuid pk
  player_id fk nullable
  source enum(steam|battlemetrics)
  captured_at timestamptz
  payload_hash text
  payload jsonb nullable
  diff_from_previous jsonb nullable
  storage_tier enum(hot|compressed|cold)
```

Rules:

- Partition snapshots by time.
- Store full first snapshot, then diffs when payloads are byte-near-identical.
- Move older raw payloads to compressed/cold storage.
- Keep normalized facts in Postgres.
- Add disk-usage alerts before snapshots fill the VPS.

---

## 6. Web GUI UX

### 6.1 Owner Cockpit

This is the first page Rob should see.

Shows:

- ingestion worker status and last heartbeat
- Steam sync health
- BattleMetrics sync health
- API quota/rate-limit health
- Discord delivery failures
- stale critical players
- online enemies count
- pending coplay suggestions
- pending moderation items
- contributors with unusual edit volume
- backup status and last restore-drill date
- disk usage and snapshot growth
- audit hash-chain verification status
- active safe mode state

Actions:

- enable safe mode
- pause alerts
- force sync a player
- retry failed notifications
- review moderation queue
- open restore bin
- view Rob todos / setup checklist

### 6.2 First-run setup wizard

Steps:

1. Create owner account through Discord OAuth.
2. Enroll owner TOTP.
3. Enter BattleMetrics token and verify scopes/capabilities.
4. Enter Steam API key and verify a test lookup.
5. Create default workspace.
6. Configure Discord bot/guild/channel permissions.
7. Configure object storage backups.
8. Run a backup and restore test into staging.
9. Set default dossier visibility and safe labels policy.
10. Add first player through quick-add.

No manual config file editing after install unless Rob chooses advanced mode.

### 6.3 Quick-add omnibox

Accepts:

- Steam profile URL
- Steam vanity URL
- SteamID64
- BattleMetrics player URL/ID
- raw player name

Behavior:

- Parse locally; do not fetch arbitrary pasted URLs.
- For URLs, allowlist host and path shape, extract ID, then call official API.
- Show resolve confidence before creating if ambiguous.
- If multiple candidates are returned, show avatar/name/server hints.
- If existing player found, jump to profile.
- If new player found, create shell record, enqueue enrichment, and show sync status.

UX states:

- resolving
- ambiguous
- already tracked
- created and syncing
- failed with retry
- source unavailable

### 6.4 Player profile

Hero card:

- avatar
- current name
- Steam/BattleMetrics links
- allegiance label + accessible icon/shape, not color alone
- threat level with text label
- ban badges
- Rust hours if public
- online/stale sync indicator
- last seen server and timestamp
- group/team badge
- last human verified timestamp

Fast actions:

- set allegiance
- set threat
- add note
- add sighting
- add tag
- watch/unwatch
- create/share dossier
- report bad intel

Sections:

- aliases timeline
- relationships lanes
- coplay suggestions
- groups
- tags
- servers
- activity heatmap
- journal/events
- media/evidence
- sync/raw data collapsed
- history/provenance

### 6.5 Mobile quick capture

Target: one-hand use mid-game.

Flow:

1. Search/select player.
2. Tap status or tag chips.
3. Optional one-line note.
4. Optional attach link/screenshot later.
5. Save with visible pending/saved/failed state.

Rules:

- Never show false saved state before server ack.
- Offline writes remain local drafts with explicit pending badge.
- Failed writes are recoverable and retryable.
- Bulk actions are hidden from quick-capture unless deliberate.

### 6.6 Relationship UX

Relationship lanes:

- Team members
- Allies
- Associates
- Enemies
- Alts / suspected alts
- Coplay suggestions

For each chip:

- avatar/name
- confidence
- source icon
- last seen together
- reason hover/tap detail
- confirm/dismiss for suggestions
- stale state if old

Avoid graph hairballs:

- default graph only shows immediate neighborhood
- cluster by group
- cap visible nodes/edges
- lazy-expand on click
- show “+43 hidden weaker edges” instead of rendering all

### 6.7 Admin/moderation UX

Admin panel:

- audit feed with filters
- moderation queue
- per-record history
- one-click revert with preview
- restore bin
- contributor management
- role changes
- session revocation
- bulk cleanup
- tag merge/delete flow
- sensitive action alerts

New contributor mode:

- optional review queue for high-impact claims
- lower default confidence
- rate limit writes
- visible attribution

Bulk operations:

- dry-run preview
- affected row count
- example affected records
- typed confirmation for destructive actions
- one-click revert created as a batch audit group

---

## 7. Discord UX

### 7.1 Dossier embed design

A Discord dossier embed must answer: “Who is this, why do we care, how fresh is it, and can I trust it?”

Embed shape:

```txt
[Avatar] CurrentName · Enemy · Threat 4/5
Freshness: BM 4m ago · Steam 2h ago · human verified 2026-06-01
Reason: Confirmed hostile on Rustoria US Main; raided us twice this wipe.
Signals: 4 aliases · 2 game bans · 3 confirmed teammates · last seen 12m ago
Confidence: High · Evidence: 2 events, 1 clip, 3 editor confirmations
Buttons: Open profile | Watch | Add note | Correct intel
```

Rules:

- No unescaped markdown from player-controlled fields.
- `allowed_mentions` parse list must be empty on every message.
- Embeds default to concise; detail lives behind buttons/pages.
- Risky labels include confidence and evidence count.
- Stale intel shows warning.
- Private fields never appear in unlisted embeds.

### 7.2 Slash commands

Lookup:

- `/dossier <player>`
- `/search <name|steamid|bm>`
- `/online`
- `/enemies`
- `/group <name>`
- `/recent`

Intel entry:

- `/track <player>`
- `/untrack <player>`
- `/note <player>`
- `/tag <player>`
- `/setstatus <player>`
- `/sighting <player>`
- `/correct <player>`

Subscriptions:

- `/watch <player|group|category>`
- `/unwatch <player|group|category>`
- `/subscriptions`
- `/alerts`

Admin:

- `/config`
- `/health`
- `/safe-mode`
- `/grant`
- `/suspend`

Rules:

- Commands that may touch DB/API call `defer()` immediately.
- Autocomplete uses cache only, never live external API calls.
- Mutations reply ephemerally.
- Read-only lookups can be public only if channel config permits it.
- DM commands require explicit permission model.

### 7.3 Alerts

Default: daily digest. Instant alerts are opt-in per subscription.

Event types:

- online
- offline
- alias_change
- new_server
- new_ban
- new_association
- dossier_update
- enemy_online
- group_online

Noise controls:

- per-player cooldown
- per-subscription cooldown
- quiet hours
- digest mode
- dedupe key per event/time window
- per-user/guild caps
- delivery failure tracking

Delivery realities:

- Prefer channel alerts for fan-out.
- Use DMs only for low-volume opt-in alerts.
- Cache DM channel IDs.
- Serialize sends through a limiter.
- On channel permission failure, notify owner and show health warning.

---

## 8. Security plan

### 8.1 Threat model

Threats:

- hostile player names causing XSS/markdown injection/pings
- malicious contributor entering stored XSS in notes
- pasted URLs causing SSRF
- Discord OAuth account compromise
- rogue admin deleting/altering data
- stale/unverified negative claims causing reputational harm
- leaked API/secrets
- object storage disclosure
- database compromise
- notification spam or Discord bot flagging
- multi-tenant leakage if expanded later

Phase 0 must create `docs/threat-model.md` before implementation begins.

### 8.2 Rendering safety

Central helpers:

- `safeDiscordText()`
- `safeDiscordEmbedField()`
- `safeHtmlText()`
- `safeMetaContent()`
- `safeMarkdownLimited()`
- `safeUrlDisplay()`

Rules:

- Discord output escapes markdown and disables mentions.
- Web uses framework escaping and sanitizes rich text with an allowlist.
- OG meta content is escaped and length-bounded.
- Avatars are cached/proxied safely.
- Notes never render raw HTML.

### 8.3 URL and media safety

- Only allow `steamcommunity.com`, `battlemetrics.com`, known image/CDN hosts configured by owner, and object-storage-hosted media.
- Resolve and block private IP ranges after redirects.
- Enforce content length, content type, timeout, and redirect limits.
- Strip metadata from uploaded images if uploads are supported.
- Store media with private bucket defaults.

### 8.4 Auth and authorization

- Discord OAuth login.
- App-side TOTP required for owner/admin.
- Step-up auth for privileged actions.
- Short session TTL for admin.
- Owner can revoke all sessions.
- Role matrix tested in CI.
- Admin cannot alter owner or other admins.
- Only owner grants/revokes admin.

Roles:

- owner
- admin
- editor
- viewer

### 8.5 Audit and anti-abuse

- Every manual write creates audit entry.
- Admin writes are also audited.
- Sensitive actions alert owner.
- Hash chain verified on schedule.
- Audit logs shipped off-box.
- Reverts create new audit entries.
- Hard purge is owner-only, delayed, and step-up gated.

### 8.6 Secrets

Secrets:

- BattleMetrics token
- Steam API key
- Discord bot token
- Discord OAuth secret
- object storage keys
- database password
- backup encryption key
- break-glass DB superuser credential

Rules:

- `.env` mode 600 on VPS or Docker secrets.
- Backup encryption key and break-glass DB credential stay offline.
- Each secret has rotation steps in `scripts/rotate-secret.md`.
- CI uses scoped deploy keys/tokens.
- Never log secrets.

---

## 9. Ingestion and sync

### 9.1 Watchlist-driven cadence

Cadence tiers:

- priority enemies: 5–15 minutes
- normal tracked: hourly
- dormant: daily
- archived/untracked: manual only

Rules:

- Compute request budget from call count × players × cadence.
- Degrade cadence gracefully when watchlist grows.
- One source outage does not block the other.
- Store per-source sync status.
- UI shows stale data plainly.

### 9.2 Identity resolution

Input resolution order:

1. SteamID64 exact
2. Steam vanity URL
3. BattleMetrics player ID
4. BattleMetrics URL
5. Steam profile URL
6. name search with disambiguation

Rules:

- SteamID64 is canonical when known.
- BM-only players are valid first-class records.
- Merge is high-risk: preview, transaction, audit, reversible unmerge.
- Alt inference is never auto-confirmed.

### 9.3 Coplay scoring

Signals:

- overlap duration
- number of distinct sessions
- recency
- server population
- repeated server co-occurrence
- time-of-day pattern

Suggestion threshold:

- minimum overlap duration
- minimum distinct sessions
- exclude/discount high-pop servers
- discount single-session coincidences
- rank by explainable score

UI must show why the suggestion exists.

### 9.4 Domain events

Ingestion produces domain events:

- player_seen_online
- player_seen_offline
- alias_changed
- ban_changed
- new_server_seen
- coplay_suggestion_created
- profile_sync_failed

Events are written to `outbox_event` and processed by notification dispatchers.

---

## 10. Dossier sharing

### 10.1 Visibility

- `private`: auth required; default for risky profiles.
- `unlisted`: long random slug; noindex; revocable; use only for sanitized summaries.

No public/indexable option.

### 10.2 Shareability gate

Before creating or rendering an unlisted dossier, check:

- no private notes included
- no doxxing fields
- no raw real-world identity fields
- no unverified serious accusation
- no stale high-impact claim without warning
- evidence/provenance present for negative labels
- owner/workspace policy permits sharing

### 10.3 Dossier page UX

Sections:

- concise summary
- freshness and confidence
- aliases
- bans/public platform facts
- server history
- relationships with provenance
- tags/playstyle
- journal/event summary
- evidence links approved for share
- correction/report bad intel action

Private full pages can include more detail. Unlisted pages show sanitized summaries only.

---

## 11. Deployment and operations

### 11.1 Recommended first deployment

Single VPS + Docker Compose:

```txt
cloudflare
  -> caddy
      -> web
      -> api
      -> worker
      -> bot
      -> postgres
      -> redis optional
```

Rules:

- Build images in CI.
- Use commit SHA tags, not `latest`.
- Deploy only after tests pass.
- One-command rollback.
- Alembic migrations are rerun-safe.
- Containers run non-root.
- Cloudflare fronts public routes.

### 11.2 Monitoring

Monitor:

- public URL uptime
- worker heartbeat
- source sync freshness
- Discord send failures
- disk usage
- snapshot growth
- Postgres health
- backup success
- restore test recency
- audit-chain verification
- Sentry/error rate
- API quota/rate-limit status

### 11.3 Backups and DR

Phase 2, not later:

- nightly encrypted `pg_dump`
- upload to object storage
- test restore into staging
- document RPO/RTO

Later hardening:

- object lock/versioning with finite retention, e.g. 30–90 days
- PITR
- immutable backup lifecycle
- restore drill schedule

---

## 12. Testing strategy

Required before opening to contributors:

- unit tests for domain rules
- integration tests with mocked Steam/BM
- golden snapshot replay tests
- role/permission matrix tests
- sanitizer tests with hostile names/notes/URLs
- Discord formatter tests with mention injection
- OG/meta escaping tests
- SSRF URL parser tests
- merge/unmerge tests
- audit revert tests
- bulk operation rollback tests
- notification dedupe/cooldown tests
- backup restore drill
- one e2e: quick-add → resolve → profile → add intel → dossier preview

CI gate blocks deploys if these fail.

---

## 13. Build phases

### Phase 0 — Product, source, and threat validation

- Create repo and docs.
- Finalize stack choice.
- Create threat model.
- Confirm BattleMetrics Premium API capabilities.
- Confirm Steam API key access and call limits.
- Confirm Discord bot permissions/intents.
- Confirm ToS/privacy constraints for redisplaying data.
- Define domain model and migrations.
- Build API clients with rate limiters.

Exit criteria:

- source capability matrix completed
- threat model completed
- no blocked ToS/legal questions for private MVP
- local dev environment boots

### Phase 1 — Core schema and ingestion

- Player records.
- Steam resolution and summaries.
- BattleMetrics resolution and server history.
- Snapshots/diffs.
- Alias capture.
- Watchlist cadence.
- Per-source sync status.
- Basic CLI/admin seed.

Exit criteria:

- add tracked player by SteamID/BM ID
- sync data persists
- staleness visible in API

### Phase 2 — Web foundation, auth, audit, backups

- Next.js app shell.
- Discord OAuth.
- Owner TOTP.
- Owner cockpit minimal.
- Quick-add omnibox.
- Player profile hero.
- Manual notes/allegiance/threat.
- Audit log for every write.
- Soft-delete.
- Encrypted off-site backups.
- Restore test.
- Output sanitization layer.

Exit criteria:

- Rob can use it solo safely
- backups are verified
- hostile input tests pass

### Phase 3 — Tags, relationships, provenance

- Tags/chips.
- Relationship lanes.
- Coplay suggestions with noise floor.
- Claim/provenance model.
- Groups.
- Confidence and stale states.

Exit criteria:

- sensitive claims show source/confidence
- coplay suggestions are explainable and dismissible

### Phase 4 — Journal, media, mobile capture, command palette

- Event timeline.
- Media/evidence links with URL safety.
- Mobile quick capture.
- Command palette.
- Bulk operations with previews.

Exit criteria:

- mid-game capture works on phone
- failed/pending writes are clear

### Phase 5 — Dossiers

- Private dossier pages.
- Unlisted sanitized pages.
- OG cards.
- Shareability gate.
- Revocation and access logging.
- Correction/report bad intel action.

Exit criteria:

- private metadata cannot leak pre-auth
- unlisted pages omit risky/private fields

### Phase 6 — Discord bot and notifications

- Slash commands.
- Autocomplete cache.
- Rich embeds.
- Buttons/modals.
- `/watch` subscriptions.
- Daily digest default.
- Instant opt-in alerts.
- Dedupe/cooldown/quiet hours.
- Delivery failure health warnings.

Exit criteria:

- Discord commands defer correctly
- allowed_mentions disabled everywhere
- alerts are not spammy

### Phase 7 — Admin/moderation and hardening

- Full moderation panel.
- Revert/restore UI.
- Contributor management.
- Role mapping.
- Sensitive action alerts.
- Hash-chain verification job.
- Off-box log shipping.
- Object-lock/PITR backup upgrade.
- Safe mode.

Exit criteria:

- contributors can be invited with controlled blast radius
- admin abuse is visible and reversible

### Phase 8 — Polish and scale

- Activity heatmaps.
- Better coplay clustering.
- Better group inference suggestions.
- Workspace install seam for future multi-tenant mode.
- Performance tuning.
- UX polish.

---

## 14. Open decisions

1. **Stack** — Python backend + Next.js + discord.py, or full TypeScript. Default remains Python/Next unless Rob chooses one-language maintenance.
2. **Dossier naming** — user-facing “player profile” may be safer than “dossier”; Discord command can remain `/dossier` if preferred.
3. **Negative labels** — final allowed vocabulary and shareability policy must be chosen before public/unlisted sharing.
4. **Contributor policy** — whether new editors require moderation review for high-impact claims.
5. **Backup retention** — choose finite object-lock retention period.
6. **Hosting** — single VPS default; PaaS only if Rob wants less ops.
7. **Legal/ToS review depth** — informal private hobby use vs. paid consultation before multi-tenant/public expansion.
8. **Language/runtime** — choose before code starts to avoid churn.

---

## 15. Definition of done for private MVP

Private MVP is done when:

- Rob can add a player from Steam/BM/name.
- Steam/BM sync works with visible freshness.
- Rob can set allegiance/threat/tags/notes.
- Audit log records every manual write.
- Backups run and restore is tested.
- Hostile names cannot ping, inject markdown links, or break HTML/OG output.
- Discord `/dossier` returns a compact, sourced, stale-aware card.
- Dossier sharing defaults to private and gates risky claims.
- Owner cockpit shows system health.
- Worker heartbeat and sync failures alert Rob.
- No contributor access is granted until moderation/revert exists.

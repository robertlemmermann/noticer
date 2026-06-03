# Noticer Architecture and Repository Layout

## Recommended architecture pattern

Use a **modular monolith backend with hexagonal ports/adapters**, plus separate web, bot, and worker processes.

This gives clear boundaries without over-splitting a small hobby system into microservices.

Core principles:

- Domain logic lives in modules, not in web controllers, Discord command handlers, or scheduled jobs.
- External services are adapters: BattleMetrics, Steam, Discord, object storage, OAuth, email/log sinks.
- Every mutation goes through a command handler that performs authorization, validation, audit logging, and event emission.
- Ingestion emits domain events through an outbox table; the dispatcher sends alerts from those events.
- Objective facts are globally shared. Subjective intel is workspace-scoped.
- Keep the initial product single-tenant, but never query subjective data without `workspace_id`.

## Proposed monorepo layout

```txt
noticer/
  README.md
  docs/
    noticer-plan.md
    audit-findings.md
    rob-todos.md
    architecture.md
    threat-model.md                 # create before implementation
    api-contract.md                 # generated/openapi notes later
  apps/
    web/                            # Next.js app: GUI + dossier SSR/OG
      app/
      components/
      features/
      lib/
      tests/
    api/                            # Backend API service
      src/
        noticer/
          main.py
          config.py
          db/
            base.py
            session.py
            migrations/             # Alembic
          modules/
            players/
            identity/
            servers/
            relationships/
            tags/
            events/
            media/
            dossiers/
            workspaces/
            users/
            audit/
            notifications/
            ingestion/
          ports/
            battlemetrics.py
            steam.py
            discord.py
            object_storage.py
            log_sink.py
          adapters/
            battlemetrics_http.py
            steam_http.py
            discord_bot_client.py
            s3_storage.py
          security/
            authz.py
            csrf.py
            sanitizer.py
            rate_limit.py
            url_safety.py
          commands/
            player_commands.py
            relationship_commands.py
            moderation_commands.py
            dossier_commands.py
          queries/
            player_queries.py
            search_queries.py
            admin_queries.py
          tests/
    worker/                         # Scheduler/ingestion/outbox dispatcher runner
      src/
      tests/
    bot/                            # Discord bot process
      src/
        commands/
        components/
        formatters/
        permission_checks/
        cache/
      tests/
  packages/
    shared-types/                   # Generated OpenAPI types or zod schemas
    ui/                             # Optional shared UI components after web stabilizes
    eslint-config/                  # Only if TS monorepo grows
  infra/
    docker/
      api.Dockerfile
      web.Dockerfile
      worker.Dockerfile
      bot.Dockerfile
    compose.yaml
    caddy/
      Caddyfile
    postgres/
      init/
    backup/
      pgdump.sh
      restore-drill.md
    cloudflare/
      waf-notes.md
  scripts/
    dev.sh
    seed.py
    verify-audit-chain.py
    replay-snapshots.py
    rotate-secret.md
  .github/
    workflows/
      ci.yml
      deploy.yml
```

## Backend module boundaries

### `players`

Owns canonical player records, Steam/BattleMetrics IDs, current name/avatar, ban state, sync state, and watchlist state.

### `identity`

Owns resolving inputs, alias history, duplicate detection, merge/unmerge, alt-account edges, and confidence scoring.

### `relationships`

Owns manual relationship edges and coplay suggestions. Coplay is never automatically promoted into a confirmed team/alliance claim.

### `tags`

Owns tag catalog, dimensions, custom tags, soft-delete/merge/delete flows, and tag permissions.

### `events`

Owns dated journal entries, sightings, raids, trades, betrayals, and evidence links.

### `media`

Owns attachments and external links. All externally rendered media goes through allowlisted/proxied access.

### `dossiers`

Owns share tokens, private/unlisted visibility, shareability gates, OG/card rendering data, revocation, and access logs.

### `notifications`

Owns subscriptions, digest/instant modes, cooldowns, quiet hours, dedupe, and delivery audit.

### `audit`

Owns append-only audit records, hash-chain verification, revert previews, and restore operations.

## Data-access rules

- Controllers and bot commands never write tables directly.
- Command handlers write and create audit records in the same transaction.
- Domain events are written to an `outbox_event` table in the same transaction as the state change.
- A dispatcher reads `outbox_event`, sends notifications, and marks events delivered or failed.
- Every workspace-scoped query must include `workspace_id`.
- Every shareable dossier read must pass through `dossier_access_policy` before rendering metadata or body.

## Suggested stack decision

Two viable paths:

### Option A — Python backend + TypeScript web + Python bot

Pros:

- FastAPI, SQLAlchemy, Alembic, async Python are stable for the API/worker.
- Python is comfortable for ingestion, data normalization, and scheduled jobs.
- Existing plan already fits this stack.

Cons:

- Two ecosystems for a small project.
- Discord bot in Python while web is TypeScript means duplicated formatting/types unless generated.

### Option B — Full TypeScript: Next.js + NestJS/Fastify + discord.js

Pros:

- One language across web, API, bot, and workers.
- Shared zod schemas/types are easier.
- Steam and BattleMetrics are REST APIs; custom clients are not hard.

Cons:

- Backend data/worker ergonomics may be less comfortable if the maintainer prefers Python.
- Needs discipline to avoid mixing domain logic into Next.js routes.

Recommended default: **Option A if implementation starts from the current plan. Option B if Rob strongly values one-language maintenance.** The plan is written to work with either; the architecture pattern matters more than the language choice.

## API style

Start with REST, not GraphQL.

REST is enough for:

- quick-add resolve
- player profile read/update
- tag operations
- relationship operations
- admin audit feed
- dossier read/share
- subscription operations

Use OpenAPI generation to produce frontend types. Add GraphQL only if the relationship graph UI becomes query-heavy enough to justify it.

## Key backend invariants

1. No untrusted name/string reaches web, Discord, or OG renderers without centralized encoding.
2. No URL pasted by a user is fetched server-side unless it matches a strict parser and allowlist.
3. No notification is sent without dedupe and `allowed_mentions: none`.
4. No sensitive claim is shareable without provenance and confidence.
5. No manual write bypasses authorization and audit logging.
6. No subjective query omits `workspace_id`.
7. No private dossier renders metadata before auth is checked.
8. No contributor can permanently delete or silently overwrite intel.

## Deployment shape

For the first real deployment:

```txt
Cloudflare
  -> Caddy reverse proxy
      -> web container
      -> api container
      -> bot container
      -> worker container
      -> postgres volume
      -> redis optional
```

Build Next.js in CI and ship slim runtime images. Keep Postgres backups off-box from the first day real data exists.

# Rob Todos — Noticer Setup and Decisions

This file contains only actions Rob needs to take. It excludes implementation tasks that can be done by the codebase later.

---

## 1. Create the GitHub repository

The ChatGPT GitHub connector can write files to an existing repository, but it cannot create a new repository from here.

Steps:

1. Open GitHub.
2. Create a new repository named `noticer` under `robertlemmermann`.
3. Choose private unless you intentionally want the plan public.
4. Do not initialize with a README if you want the local package imported cleanly; either choice is workable.
5. After creating it, return to ChatGPT and say: `Push the Noticer plan to robertlemmermann/noticer`.

---

## 2. Pick the implementation stack

Choose one before code starts.

### Option A — Python backend + Next.js web + discord.py bot

Choose this if you want to follow the current plan with minimal changes.

Rob action:

1. Confirm: `Use Python backend + Next.js + discord.py`.

### Option B — Full TypeScript

Choose this if you want one language across web, API, worker, and Discord bot.

Rob action:

1. Confirm: `Use full TypeScript for Noticer`.

Recommendation from the audit: either works. The architecture pattern matters more than the language. Full TypeScript reduces context switching; Python/Next follows the current plan more directly.

---

## 3. Create or confirm BattleMetrics access

Steps:

1. Open BattleMetrics developer/account area.
2. Create a personal access token.
3. Confirm Premium is active.
4. Record which scopes the token has.
5. Do not paste the token into chat unless you intentionally want to use it in a live environment; store it in the app/VPS secret store later.
6. During Phase 0, run a capability test for:
   - deep session history
   - extended coplay history
   - player flags through API
   - player notes through API
   - rate-limit headers / 429 behavior

Decision to record:

```txt
BattleMetrics Premium API gives extended session/coplay through token: yes/no/partial
BattleMetrics flags through token: yes/no
BattleMetrics notes through token: yes/no
```

---

## 4. Create or confirm Steam Web API key

Steps:

1. Open Steam Web API key page.
2. Create/confirm an API key.
3. Store it as a secret later.
4. Phase 0 test calls:
   - ResolveVanityURL
   - GetPlayerSummaries
   - GetPlayerBans
   - GetFriendList on public/private profile
   - GetOwnedGames on public/private profile

Decision to record:

```txt
Steam API key available: yes/no
Steam friend/playtime fields acceptable for MVP: yes/no
```

---

## 5. Decide privacy boundaries now

Answer these before implementation.

1. Should Noticer store Steam public `realname` at all?
   - Safer default: no.
2. Should unlisted dossiers ever show negative labels?
   - Safer default: only with high confidence and evidence.
3. Should viewers be able to create unlisted share links?
   - Safer default: no; editor+ only, or admin-only at first.
4. Should `cheater-suspect` be an allowed tag?
   - Safer default: yes only as a private/internal tag with evidence required before sharing.
5. Should player correction/report links be enabled on unlisted pages?
   - Safer default: yes.

Write decisions:

```txt
Store Steam realname: no / yes
Unlisted negative labels: no / yes with evidence / yes unrestricted
Who can create unlisted links: owner/admin/editor/viewer
Cheater-suspect policy: private only / shareable with evidence / disabled
Correction workflow: enabled / disabled
```

---

## 6. Buy or choose a domain

Steps:

1. Pick a domain or subdomain.
2. Point DNS to Cloudflare.
3. Plan subdomains:
   - `noticer.<domain>` or `app.<domain>` for the web app
   - `api.<domain>` for backend API if using separate host routing
4. Keep dossiers under `/d/<slug>` on the web app domain.

Decision to record:

```txt
Domain:
App host:
API host:
```

---

## 7. Choose hosting

Default: single VPS with Docker Compose.

Steps:

1. Pick provider: Hetzner, DigitalOcean, Vultr, or similar.
2. Start with roughly 2 vCPU / 4 GB RAM.
3. Enable backups/snapshots at provider level if affordable.
4. Lock down SSH:
   - SSH key only
   - disable password login
   - non-root deploy user
5. Install Docker and Docker Compose later.

Decision to record:

```txt
VPS provider:
Region:
Monthly budget:
```

---

## 8. Create Discord bot application

Steps:

1. Open Discord Developer Portal.
2. Create application named `Noticer`.
3. Create bot user.
4. Save bot token in a secret manager later.
5. Enable needed intents only after confirming implementation needs:
   - Guilds
   - Guild Members if role mapping/member resolution requires it
   - No message-content intent for slash-command-only design
6. Invite bot to your Discord server with least-needed permissions:
   - Send Messages
   - Embed Links
   - Use Slash Commands
   - Read Message History if needed for interaction context
   - Attach Files only if using generated images
7. Create channels:
   - `#noticer-alerts`
   - `#noticer-digests`
   - optional private admin/mod channel

Decision to record:

```txt
Discord guild ID:
Alert channel:
Digest channel:
Admin channel:
```

---

## 9. Decide Discord alert defaults

Recommended defaults:

- Daily digest on by default.
- Instant alerts opt-in only.
- DM alerts opt-in only.
- Channel alerts for shared enemy watch.
- Online alert cooldown: 5 minutes.
- Quiet hours enabled per user.

Rob decision:

```txt
Default alert mode: daily digest / instant
Instant online cooldown minutes:
Allow DM alerts: yes/no
Alert channel enabled: yes/no
```

---

## 10. Configure object storage for backups

Steps:

1. Create Backblaze B2, AWS S3, or compatible object storage bucket.
2. Enable versioning if available.
3. Choose immutable/object-lock retention later, after basic backups work.
4. Create restricted access key for backup upload only.
5. Decide retention:
   - recommended finite immutable retention: 30–90 days
6. Store backup encryption key offline, not in app `.env`.

Decision to record:

```txt
Object storage provider:
Bucket name:
Immutable retention days:
Backup encryption key storage location:
```

---

## 11. Create owner MFA plan

Steps:

1. Decide which TOTP app to use.
2. Enroll owner TOTP during first-run setup.
3. Save recovery codes offline.
4. Require TOTP for admin/owner actions.

Decision to record:

```txt
TOTP app:
Recovery code storage location:
```

---

## 12. Decide contributor policy

Recommended initial policy:

- No contributors until Phase 7 moderation UI is live.
- Editor role can add intel but cannot delete old intel.
- New editors have high-impact claims queued for review until trusted.
- Admin role is granted manually by owner only.

Rob decision:

```txt
Initial contributors allowed before Phase 7: no / yes
High-impact claim review for new editors: yes / no
Who can create unlisted dossiers: owner/admin/editor/viewer
```

---

## 13. Decide allowed negative labels

Create a controlled vocabulary before sharing is enabled.

Suggested internal labels:

- hostile
- raider
- offline-raider
- doorcamper
- roofcamper
- scammer
- suspected cheater
- confirmed teammate
- ally
- neutral
- peaceful

Rules to decide:

1. Which labels are internal-only?
2. Which labels can appear in unlisted dossiers?
3. What evidence level is required?

Write decisions:

```txt
Internal-only labels:
Shareable labels:
Evidence required for suspected cheater:
Evidence required for scammer:
Evidence required for hostile/enemy:
```

---

## 14. Prepare incident/safe-mode policy

Answer these before inviting contributors.

1. Who can enable safe mode?
2. What safe mode disables?
3. Who gets alerted on suspicious admin/contributor actions?
4. How long to retain audit logs and backups?

Recommended safe mode disables:

- unlisted dossier creation
- outbound instant alerts
- Discord public lookup responses
- media rendering from external URLs
- contributor writes except owner/admin

Write decisions:

```txt
Safe mode can be enabled by:
Safe mode disables:
Owner alert destination:
```

---

## 15. Phase 0 acceptance checklist

Before writing real app code, verify these are true:

- [ ] GitHub repo `robertlemmermann/noticer` exists.
- [ ] Stack is chosen.
- [ ] BattleMetrics token exists.
- [ ] BattleMetrics API capability spike is complete.
- [ ] Steam API key exists.
- [ ] Discord bot application exists.
- [ ] Privacy boundary decisions are written down.
- [ ] Negative label vocabulary is chosen.
- [ ] Domain/hosting direction is chosen.
- [ ] Backup object storage is chosen.
- [ ] Owner MFA plan is chosen.
- [ ] Contributor policy is chosen.
- [ ] Threat model file exists.

---

## 16. What to ask ChatGPT/Codex next

After creating the GitHub repo, use:

```txt
Push the local Noticer plan files to robertlemmermann/noticer.
```

After Phase 0 decisions are filled in, use:

```txt
Create the initial Noticer monorepo skeleton from docs/architecture.md, using the chosen stack, with no real secrets committed.
```

After the skeleton exists, use:

```txt
Implement Phase 0 source capability spike scripts for BattleMetrics and Steam, with mocked tests and clear README instructions.
```

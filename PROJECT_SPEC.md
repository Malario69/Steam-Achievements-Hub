# Steam Achievements Hub — Project Specification

## 1. Product vision

NextUnlock is a web application for achievement hunters. It combines official Steam player/achievement data fetched on demand with persistent NextUnlock-owned roadmap data and external hunting metadata to answer core hunting questions:

1. What games and achievements does the user currently have?
2. Which achievements are open right now?
3. How can those open achievements be turned into an efficient hunting roadmap?
4. What should the user hunt next?

The application should be useful both as a personal dashboard and, later, as a multi-user public web service.

---

## 2. Core product principles

- Steam facts and locally derived data must be distinguishable.
- User-specific Steam state is not a permanent historical datastore by default; persist only what a concrete NextUnlock feature requires.
- A game can be 100% complete locally while not being eligible for Steam profile showcases.
- External metadata such as difficulty and completion time must include source/provenance and freshness.
- Recommendations should initially be explainable and deterministic rather than opaque AI output.
- Steam/API secrets must never reach the browser or repository.
- Synchronization must be idempotent and safe to retry.

---

## 3. Version 1 scope

### 3.1 Authentication

- Sign in with Steam OpenID.
- Store the verified SteamID64 as the user's external identity.
- Do not collect Steam usernames/passwords.
- Do not request end users to provide their Steam Web API key.
- Use one server-side Steam Web API key configured by the deployment operator.

### 3.2 Library synchronization

A user can press a `Synchronize Steam` button.

The sync process should:

1. Resolve the signed-in Steam user.
2. Retrieve owned games visible to the application.
3. Upsert global game records by Steam AppID.
4. Retrieve/cache global game and achievement schema data where useful.
5. Retrieve the user's current achievement state on demand.
6. Calculate completion statistics for the current view.
7. Persist only NextUnlock-owned data required by features such as roadmaps.
8. Do not persist broad historical library/playtime/achievement snapshots unless an explicitly approved feature requires a minimal dataset.

The web request should enqueue work; large Steam libraries must not be processed synchronously inside one HTTP request.

### 3.3 Library dashboard

Show at minimum:

- games owned
- games with achievements
- games started
- untouched games
- total achievements available
- total achievements unlocked
- overall completion percentage
- perfect games according to local completion
- perfect games eligible for Steam profile/showcase counting
- games marked as profile-feature-limited
- current completion status
- current roadmap status where available
- data freshness / last successful refresh

### 3.4 Game detail page

Each game should show:

- title and Steam AppID
- cover/header image where available
- playtime
- unlocked / total achievements
- completion percentage
- achievement list
- global achievement rarity where available
- hunting completion-time estimate
- hunting difficulty estimate
- genres/tags/categories
- hunting flags such as multiplayer, online, missable, grind, collectibles, difficulty-specific, DLC, NG+
- Steam profile eligibility state
- recommended guides
- saved NextUnlock roadmap/progress where available
- source/provenance for enriched metadata

### 3.5 Historical Steam-state features

The default architecture does **not** persist broad historical user Steam state.

If a future feature such as long-term lost-perfect tracking requires historical Steam-derived user data, it must first receive an explicit architecture/privacy review. The implementation must define the minimum required fields, retention, deletion behavior and user impact before adding persistence.

Roadmap refresh should prefer comparing saved NextUnlock roadmap state with current Steam data fetched on demand rather than building a general-purpose Steam history database.

### 3.6 Profile eligibility

Treat profile/showcase eligibility independently from completion percentage.

Suggested normalized state:

- `ELIGIBLE`
- `PROFILE_FEATURES_LIMITED`
- `UNKNOWN`

User-level privacy/private-game limitations are separate concerns and must not be conflated with an app-level profile-features-limited state.

Store `profileEligibilityCheckedAt` so this metadata can be refreshed independently from player achievement state.

### 3.7 Hunting metadata

External/curated metadata must be stored through a provider abstraction rather than hard-coded to one website.

Suggested fields:

- `completionTimeMinMinutes`
- `completionTimeMaxMinutes`
- `difficultyScore` (normalized 1–10)
- `difficultyConfidence`
- hunting flags/categories
- source/provider
- source URL
- retrieved timestamp
- optional raw/provider-specific reference

Sources may later include community sites, curated data, manually maintained entries, and locally calculated scores.

### 3.8 Guide discovery

Create a guide provider abstraction.

The initial Steam guide provider should support:

- achievement-related guide search for a Steam AppID
- ranking by useful quality signals where available
- title, URL/reference, author/display information where available
- vote/rating metadata where available
- cached results with freshness timestamps

Search terms may include `achievement`, `achievements`, `100%`, `completion`, `missable`, and `walkthrough`.

### 3.9 Recommendations

Two recommendation modes:

1. `My Library` — recommend owned games to hunt next.
2. `Discover` — recommend games not currently owned.

Initial recommendation scoring must be deterministic and explainable.

Potential inputs:

- genre/tag similarity
- expected completion time
- difficulty
- profile eligibility
- review quality
- price for store recommendations
- whether online achievements are present
- whether missables are present
- user's historic completion behavior
- user's explicit filters

Each recommendation should include a short explanation of why it ranked highly.

---

## 4. Proposed architecture

### 4.1 Monorepo

```text
apps/
  web/             Next.js web application and server-side API surface
  worker/          Background synchronization/enrichment worker
packages/
  database/        Prisma schema and DB access
  steam-api/       Official Steam Web API client
  steam-openid/    Steam OpenID integration
  steam-store/     Steam Store metadata adapter
  steam-community/ Steam Community/guide/profile-feature adapter
  achievement-engine/
  recommendation-engine/
  providers/       Hunting metadata provider contracts
  shared/          Shared schemas, types, utilities
```

### 4.2 Infrastructure

- PostgreSQL: primary persistent store
- Redis: job queue/cache where appropriate
- BullMQ: background jobs
- Docker Compose: local development
- GitHub Actions: lint, typecheck, test, build, security checks

### 4.3 Application boundaries

The browser must never call privileged Steam endpoints with a secret key.

```text
Browser
  -> Next.js application
      -> authenticated server route/action
          -> queue job
              -> worker
                  -> Steam/external providers
                  -> PostgreSQL
```

---

## 5. Initial domain model

This is conceptual; exact Prisma naming may change during implementation.

### User

- id
- steamId64 (unique)
- createdAt
- lastLoginAt

### Game

- id
- steamAppId (unique)
- name
- hasAchievements
- profileEligibility
- profileEligibilityCheckedAt
- createdAt
- updatedAt

### AchievementDefinition

- id
- gameId
- apiName
- displayName
- description
- hidden
- iconUrl
- iconGrayUrl

Unique key: `(gameId, apiName)`.

### AchievementSchemaSnapshot

- id
- gameId
- schemaHash
- achievementCount
- capturedAt

### SyncRun

- id
- userId
- status
- trigger (`MANUAL`, later possibly `SCHEDULED`)
- startedAt
- finishedAt
- errorSummary
- gamesProcessed
- gamesTotal

### HuntingMetadata

- id
- gameId
- provider
- completionTimeMinMinutes
- completionTimeMaxMinutes
- difficultyScore
- confidence
- sourceUrl
- retrievedAt

### HuntingFlag

- id
- gameId
- type
- value
- provider
- confidence
- retrievedAt

### Guide

- id
- gameId
- provider
- externalId
- title
- url
- score/rating fields
- retrievedAt

---

## 6. Achievement schema hashing

For change detection, create a canonical representation containing relevant stable achievement definition fields, sort by achievement API name, serialize deterministically, and hash with SHA-256.

At minimum, a count increase combined with a changed schema hash can produce `ACHIEVEMENTS_ADDED`.

Do not rely only on the achievement count because definitions can change without count changes.

---

## 7. Sync correctness requirements

- One user should not have two conflicting active full syncs.
- Jobs must be safe to retry.
- A failed sync must not replace the user's `lastSuccessfulSyncAt`.
- Partial failures must be recorded.
- Previous successful snapshots must stay intact.
- External provider failures must not invalidate valid Steam achievement data.
- API rate limits and transient errors require bounded retry/backoff.
- Never log credentials or secret-bearing URLs.

---

## 8. Security baseline

The project follows a BSI-oriented security baseline, without claiming certification.

Required practices:

- secrets only in server-side environment/secret storage
- `.env` ignored by Git
- committed `.env.example` contains names only, never credentials
- secure HTTP-only sessions/cookies
- CSRF protection for state-changing browser requests
- restrictive security headers and CSP
- validation at every trust boundary
- authorization checks on every user-scoped API operation
- rate limiting for authentication and expensive sync actions
- SSRF-safe handling of external URLs
- parameterized DB access through Prisma
- dependency and code scanning in CI
- no sensitive values in logs
- least-privilege database/runtime identities
- encrypted transport
- documented backup/restore process before production
- data export/deletion capability before multi-user public launch
- privacy documentation before production

---

## 9. Secret/environment names

Expected names only; values must never be committed.

```text
DATABASE_URL=
REDIS_URL=
STEAM_WEB_API_KEY=
APP_BASE_URL=
SESSION_SECRET=
```

Additional provider secrets must follow the same rule.

---

## 10. Development milestones

### Milestone 0 — Foundation

- monorepo scaffold
- TypeScript/lint/format/test conventions
- Docker development stack
- PostgreSQL + Prisma
- Redis + worker
- environment validation
- CI

### Milestone 1 — Steam identity and library

- Steam OpenID sign-in
- user record
- owned-games sync
- library page
- manual sync job/progress

### Milestone 2 — Achievements

- achievement schema cache
- player achievements
- global rarity
- completion calculations
- game detail page

### Milestone 3 — Roadmaps

- roadmap generation from currently open achievements
- persistent roadmap storage
- manual progress/checkmarks
- refresh detection without silently overwriting user progress
- minimal snapshot data only where required for roadmap refresh

### Milestone 4 — Profile eligibility

- app profile-feature state
- Steam-eligible vs local perfects
- refresh strategy

### Milestone 5 — Hunting intelligence

- provider abstraction
- completion time
- difficulty
- hunting flags/categories
- confidence/provenance

### Milestone 6 — Guides

- Steam guide provider
- ranking/cache
- game-detail integration

### Milestone 7 — Recommendations

- owned-library recommendation engine
- filters/explanations
- store discovery recommendations

### Milestone 8 — Production hardening and security gate

- privacy/export/delete flows
- observability
- backup/restore
- stronger rate limiting
- security review/threat model
- deployment documentation

---

## 11. Explicit non-goals for the first implementation

- automatic continuous monitoring
- storing end-user Steam API keys
- scraping every external hunting site directly in core business logic
- AI-generated difficulty as the primary source of truth
- mobile-native apps
- social/friend leaderboards
- automatic achievement unlocking or game modification

---

## 12. Definition of success for the first usable release

A user can sign in with Steam, inspect currently accessible library/achievement data, create and persist an efficient achievement roadmap, keep manual roadmap progress, and refresh current Steam state without silently overwriting saved roadmap progress.


---

## 13. Binding security and data-minimization decision

The documents in `docs/security/` are the authoritative security baseline for authentication, data persistence, secrets, deployment, backups, security testing and incident response.

For the current architecture, the only Steam user datum intentionally persisted long-term is the verified SteamID64. User-specific library, playtime and achievement state are fetched on demand or held only in short-lived caches unless a future feature receives an explicit documented architecture/privacy approval for additional minimal persistence.

If this specification conflicts with `docs/security/`, the stricter and more recent documented security/data-minimization decision must be resolved before implementation rather than silently choosing the broader data model.

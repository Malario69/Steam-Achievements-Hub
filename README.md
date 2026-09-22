# NextUnlock

A web application for Steam achievement hunters: current library/achievement insights, persistent hunting roadmaps, hunting metadata, guide discovery, and personalized recommendations with a data-minimizing security model.

## Project goals

- Sign in with Steam without collecting Steam credentials.
- Import the user's owned games and achievement progress.
- Distinguish local 100% completion from Steam profile eligibility.
- Build and persist efficient hunting roadmaps without maintaining an unnecessary permanent copy of a user's Steam state.
- Enrich games with completion-time, difficulty, genres, achievement categories, and guide links.
- Recommend games to hunt from both the user's library and the Steam catalog.
- Keep all privileged Steam/API access server-side.
- Follow a security-first, BSI-oriented development approach.

## Planned stack

- Next.js + TypeScript
- PostgreSQL
- Prisma
- Redis + BullMQ for synchronization jobs
- Steam OpenID for sign-in
- Steam Web API and Steam Store/Community adapters
- Docker for local/self-hosted deployment
- GitHub Actions for CI

## Current status

Project foundation and architecture are being specified. See `PROJECT_SPEC.md`, `CLAUDE.md`, and `docs/security/`.

## Security rule

Never commit real API keys, session secrets, passwords, tokens, or production credentials. Secrets belong only in runtime secret storage. The current data-minimization baseline persists SteamID64 as the only long-term Steam user datum; see `docs/security/` for the binding rules.

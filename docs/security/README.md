# NextUnlock security documentation

This directory is the authoritative security and privacy baseline for NextUnlock.

Read these documents before implementing authentication, Steam integration, persistence, deployment, backups, or production releases:

- [SECURITY-ARCHITECTURE.md](./SECURITY-ARCHITECTURE.md) — trust boundaries, sessions, secrets, database and Steam integration
- [DATA-INVENTORY.md](./DATA-INVENTORY.md) — what may and may not be persisted
- [DEPLOYMENT-CHECKLIST.md](./DEPLOYMENT-CHECKLIST.md) — mandatory server and production checks
- [SECURITY-TESTING.md](./SECURITY-TESTING.md) — staging security gate and retest process
- [INCIDENT-RESPONSE.md](./INCIDENT-RESPONSE.md) — actions for key, session, database or server compromise

## Core rule

Persist as little Steam user data as possible. For the current design, the only Steam user datum intentionally persisted long-term is the verified SteamID64. Steam credentials, Steam login sessions, end-user API keys, library data, playtime and achievement history are not long-term application data.

NextUnlock-created data such as roadmaps, roadmap progress and settings may be persisted and linked through an internal user ID.

Any future feature that requires additional persistent Steam-derived user data must be treated as an explicit architecture/privacy decision and documented before implementation.

## Security is a release gate

Production is not allowed merely because the application works. Relevant deployment, backup, authorization and security-test checks must pass first.

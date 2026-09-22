# Security architecture

## 1. Security goal

A compromise of NextUnlock should have a deliberately limited blast radius. NextUnlock must not hold Steam credentials or Steam authorization material that could be used to take over a user's Steam account.

No system can be made absolutely secure. The design therefore focuses on data minimization, least privilege, strong isolation, recoverability and repeatable security testing.

## 2. Steam authentication

- Authentication uses Steam OpenID.
- The OpenID response must be validated server-side.
- The verified SteamID64 is the external identity.
- Never collect or store Steam passwords, Steam Guard codes or Steam login-session material.
- Never trust a SteamID64 supplied by the browser for an authenticated user-scoped action. Resolve it server-side from the authenticated NextUnlock session.
- A user's Steam privacy settings still govern what Steam exposes. OpenID login does not grant NextUnlock extra access to private Steam data.

Flow:

```text
Browser -> Steam OpenID -> NextUnlock backend
                           |
                           -> verified SteamID64
                           -> internal user_id
```

## 3. Persistent identity

Use an internal identifier as the primary relation key.

```text
users
- id (UUID)
- steam_id64 (unique)
- created_at
- last_login_at
```

All NextUnlock-owned data references `users.id`, not SteamID64 directly.

## 4. Steam Web API

- Use Valve's central Steam Web API.
- A project-dedicated Steam account should own the normal Steam Web API key rather than a private personal Steam account.
- The API key is an application secret, not a user credential.
- Browser code never receives the Steam API key.
- Steam requests are made by the backend/worker only.
- Avoid secret-bearing query strings where a supported header is available.
- Do not perform a full per-game achievement scan on every login.
- Fetch user-specific Steam data on demand and cache only as needed with a short TTL.
- Global game/achievement definitions may be stored centrally because they are not user-specific.

## 5. Session model

Use opaque server-side sessions.

Browser:
- cryptographically random session token
- cookie flags in production: `Secure`, `HttpOnly`, `SameSite=Lax`
- appropriate `Path` and expiry
- never store auth tokens in localStorage

Server-side session record:

```text
sessions
- token_digest
- user_id
- created_at
- expires_at
- revoked_at (optional)
```

The stored digest should be an HMAC of the random token using a backend-only `session_secret` (or a framework-native equivalent with at least the same security properties). The raw session token is not stored server-side.

Logout/revocation must invalidate the server-side session.

## 6. Production secrets

Initial secrets:

```text
steam_api_key
database_password
session_secret
```

Rules:
- never commit them to Git
- never put them in client bundles
- never bake them into Docker images
- never store them as normal application rows
- never log them
- do not expose them in error messages
- production secrets are made available only to the services that require them

For the planned VPS + Docker Compose deployment, use protected host-side secret files, for example:

```text
/etc/nextunlock/secrets/production/
  steam_api_key
  database_password
  session_secret
```

with restrictive ownership/permissions, then mount/provide them to the required backend service via Docker secrets/read-only files.

Local development may use an ignored `.env`. Production should not depend on a committed or image-baked `.env`.

Vault/HSM/KMS are intentionally not required for V1. Revisit a managed secret manager when the infrastructure, team or secret count justifies it.

## 7. Database isolation

- PostgreSQL is not publicly exposed to the Internet.
- Application access occurs through an internal/private Docker network or loopback/private interface.
- Use a dedicated least-privilege application database role.
- Do not run the app as the PostgreSQL superuser.
- Use migrations and parameterized/ORM database access.
- Authorization remains an application-layer requirement; database secrecy does not replace per-user access control.

## 8. Network and host baseline

- Public entry points should be limited to required services, normally HTTPS and tightly controlled SSH.
- Use TLS/HTTPS for browser traffic and upstream Steam requests.
- Keep OS, Docker/runtime and application dependencies patched.
- Prefer SSH keys; disable password-based root login.
- Firewall unnecessary ports.
- Run services with least privilege and avoid privileged containers.
- Do not mount the Docker socket into application containers.

## 9. Backup security

Provider snapshots are not the only backup strategy.

Before production:
- create database backups explicitly
- encrypt application/database backups client-side, e.g. `pg_dump` + Restic
- send encrypted backups to a separate backup target such as Hetzner Object Storage/Storage Box
- protect backup credentials as secrets
- test restore procedures, not only backup creation
- keep backups logically separate from the live application where practical

Full-disk encryption may be considered, but it does not replace application isolation, secret handling, access controls or encrypted backups. Once a running server is compromised, disk encryption does not protect secrets already available to the running system.

## 10. Authorization invariant

Every user-scoped route must derive the acting user from the authenticated session.

Bad:

```text
POST /api/roadmap
steamId64=<browser supplied value>
```

Good:

```text
session -> internal user_id -> server-side steam_id64
```

A user must never be able to read or mutate another user's roadmap by changing an ID.

## 11. Logging

Do not log:
- API keys
- DB passwords
- session tokens/cookies
- Authorization headers
- secret-bearing URLs
- full sensitive request bodies

Log enough metadata for diagnosis and incident response without creating a second sensitive-data store.

## 12. Compromise assumptions

A database-only leak, API-key leak and full server compromise are different incidents.

A full server/root compromise is high severity because the attacker may control runtime code and access secrets in use. Treat the host as untrusted after such an event: rebuild from a known-good image, rotate secrets, invalidate sessions and investigate logs.

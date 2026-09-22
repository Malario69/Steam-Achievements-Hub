# Production deployment security checklist

This checklist is a release gate. Do not deploy to production with unresolved critical items.

## Host and network

- [ ] Supported/patched Linux distribution
- [ ] Security updates applied
- [ ] SSH key authentication configured
- [ ] Password-based root login disabled
- [ ] Firewall enabled
- [ ] Only required public ports exposed
- [ ] PostgreSQL port is not publicly reachable
- [ ] Redis, if used, is not publicly reachable
- [ ] Docker socket is not exposed to application containers
- [ ] Containers do not run privileged unless explicitly justified
- [ ] TLS/HTTPS configured and verified
- [ ] HTTP redirects to HTTPS where appropriate

## Secrets

- [ ] Separate project-dedicated Steam account/API key
- [ ] `steam_api_key` absent from repository history/current tree
- [ ] `database_password` absent from repository
- [ ] `session_secret` absent from repository
- [ ] Production secrets stored as protected runtime/Docker secrets
- [ ] Secret files have restrictive owner/permissions
- [ ] Frontend/client bundle contains no secrets
- [ ] Docker images contain no secrets
- [ ] Logs contain no secrets/cookies/auth headers
- [ ] Secret rotation procedure documented

## Database

- [ ] Dedicated least-privilege DB role
- [ ] Application is not using PostgreSQL superuser
- [ ] Database accessible only from required internal services
- [ ] Migrations reviewed
- [ ] No raw Steam user-history tables were added contrary to data policy
- [ ] Authorization tests pass for all user-owned records

## Authentication and sessions

- [ ] Steam OpenID validation occurs server-side
- [ ] SteamID64 for user actions is derived from authenticated session, not trusted from browser input
- [ ] Session tokens are cryptographically random
- [ ] Cookies use `Secure` in production
- [ ] Cookies use `HttpOnly`
- [ ] Cookies use an appropriate `SameSite` policy (baseline: `Lax`)
- [ ] Server stores only token digest/HMAC, not raw session token
- [ ] Session expiry enforced
- [ ] Logout/revocation works
- [ ] CSRF protections are appropriate for state-changing endpoints

## Steam/API use

- [ ] Steam Web API calls occur server-side only
- [ ] Rate limiting/backoff implemented where necessary
- [ ] Expensive Steam sync/refresh actions are abuse-limited
- [ ] No full achievement scan is triggered unnecessarily on each login
- [ ] User-specific Steam responses are on-demand/short-lived cache only unless explicitly approved
- [ ] Global game/achievement data is stored separately from user data

## Backups and recovery

- [ ] Database backup process exists
- [ ] Backups are encrypted client-side (e.g. Restic)
- [ ] Backup target is separate from live DB storage
- [ ] Backup credentials are protected secrets
- [ ] Retention policy documented
- [ ] Restore has been tested successfully
- [ ] Recovery instructions documented

## Application security

- [ ] Input validation at trust boundaries
- [ ] Authorization/IDOR tests pass
- [ ] SQL/command injection review complete
- [ ] XSS/CSP/security-header review complete
- [ ] Rate-limit/abuse tests complete
- [ ] Dependency/CVE scan reviewed
- [ ] Secret scan reviewed
- [ ] No debug endpoints or stack traces exposed
- [ ] Error messages do not reveal internals

## Release gate

- [ ] Staging security review complete
- [ ] Automated scan complete
- [ ] Manual auth/API/access-control tests complete
- [ ] Critical/High findings fixed or explicitly blocked from release
- [ ] Fixes retested
- [ ] Incident-response document reviewed

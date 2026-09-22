# Security testing and release gate

## 1. Principle

Security testing is continuous and is repeated before public production releases. Passing a scanner does not prove the application is secure.

Use OWASP ASVS/WSTG concepts as a practical reference baseline where applicable.

## 2. Required pre-production flow

```text
feature/code complete
 -> staging deploy
 -> code/config security review
 -> automated scans
 -> manual auth/API/access-control tests
 -> fix findings
 -> retest
 -> production
```

## 3. Manual tests that matter most for NextUnlock

### Authorization / IDOR

Create at least two test users.

Verify User B cannot:
- read User A's roadmap by changing a roadmap ID
- update/delete User A's roadmap
- mark User A's roadmap steps
- access any user-scoped endpoint by changing a user ID

The server must derive ownership from the authenticated session.

### Steam identity handling

Verify:
- forged/unvalidated OpenID callbacks are rejected
- user-scoped Steam calls do not trust a browser-supplied SteamID64
- private/unavailable Steam data produces an explicit unavailable/privacy state, not a fabricated zero state

### Sessions

Verify:
- invalid/random session tokens fail
- expired sessions fail
- logout invalidates the session
- revoked sessions fail
- cookies have required flags
- session token is not present in logs or client storage
- session fixation/reuse scenarios are handled appropriately

### Input/injection

Test:
- SQL injection
- command injection where any process execution exists
- XSS through names/titles/provider metadata
- oversized/malformed JSON/input
- unsafe URLs/SSRF if external URL fetching exists
- path traversal if file operations are introduced

### Abuse/rate limiting

Test:
- repeated login/callback abuse
- repeated Steam refresh/sync requests
- endpoints that could exhaust Steam API quota
- resource-intensive roadmap generation
- queue/job duplication if workers are introduced

## 4. Automated checks

CI/staging should include, as relevant:
- dependency vulnerability scanning
- static analysis/code scanning
- secret scanning
- lint/typecheck/tests
- container/image vulnerability scanning
- infrastructure/config checks
- dynamic web scanning against staging (e.g. OWASP ZAP)

Automated tools supplement manual review; they do not replace it.

## 5. Server/config review

Check:
- exposed ports
- SSH settings
- firewall
- PostgreSQL/Redis exposure
- Docker privileges/mounts
- TLS configuration
- HTTP security headers
- filesystem permissions
- secret locations
- backup permissions
- logging configuration

## 6. Finding format

Document findings as:

```text
Severity: Critical / High / Medium / Low / Informational
Location:
Description:
Attack scenario:
Impact:
Fix:
Retest result:
```

Do not use severity labels as a substitute for explaining actual impact.

## 7. Release criteria

- Critical findings block release.
- High findings normally block release until fixed/retested.
- Medium/Low findings require an explicit documented disposition and follow-up.
- Every security fix gets a retest.
- Do not test destructively against production when staging can represent the behavior safely.

## 8. Recurring review

Repeat relevant tests after:
- authentication/session changes
- new user-scoped endpoints
- database permission/schema changes
- new external providers
- deployment/network changes
- secret-management changes
- major dependency/framework upgrades

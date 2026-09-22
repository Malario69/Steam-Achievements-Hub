# Incident response

This document is an operational checklist, not a substitute for legal advice.

## 1. First principles

If compromise is suspected:
1. preserve evidence/logs where practical
2. contain the incident
3. determine scope
4. rotate/revoke affected credentials
5. restore from a trusted state
6. assess user/data impact
7. document decisions and timeline
8. perform required legal/privacy notification assessment

Do not assume that changing one key makes a fully compromised server trustworthy again.

## 2. Steam Web API key leak

- Revoke/replace the compromised key
- Update the production runtime secret
- Restart/reload only the services that require it
- Review logs for misuse/abnormal request volume
- Check repository history, CI logs, container images and application logs for the leak source
- Remove the root cause before considering the incident closed

A normal Steam Web API key is not a Steam password/session, but misuse and quota/terms impact are still possible.

## 3. Session-cookie leak

- Revoke the affected session(s)
- If scope is uncertain, invalidate all NextUnlock sessions
- Investigate how the raw token was exposed (XSS, logs, transport, browser compromise, etc.)
- Fix root cause and retest

A NextUnlock session must not be usable as a Steam login session.

## 4. Database credential leak

- Rotate the DB password/credential
- Confirm PostgreSQL was not publicly reachable
- Review database/network logs where available
- Determine whether data was read, modified or deleted
- Verify backups before destructive recovery actions
- Reissue application credential with least privilege

## 5. Database data leak

Assess what was exposed.

For the intended current design, persistent Steam user data should be limited to SteamID64 plus NextUnlock-owned data such as roadmaps/settings/session digests.

Actions:
- contain access path
- preserve evidence
- determine affected users/time window
- verify whether session material was usable
- evaluate legal/privacy notification duties
- notify affected users where required/appropriate
- fix and retest

## 6. Full server/root compromise

Treat the host as untrusted.

- Isolate the compromised server
- Preserve relevant logs/evidence if feasible
- Provision a clean server from known-good sources
- Patch the exploited weakness before reopening service
- Rotate all production secrets, including:
  - Steam API key
  - database credentials
  - session secret
  - backup credentials
  - SSH/deployment credentials that may have been exposed
- Invalidate all NextUnlock sessions
- Restore only validated data/backups
- Review application/repository/CI/CD for persistence or secret exposure
- perform security retest before production traffic returns

Do not simply clean a few files on a root-compromised host and continue trusting it.

## 7. Backup compromise

Because backups should be client-side encrypted:
- rotate backup-access credentials
- assess whether encryption keys/passwords were exposed separately
- verify backup integrity and retention
- replace exposed backup encryption material if necessary
- re-establish trusted backup sets

## 8. Data-protection assessment

A security incident is not automatically the same as a reportable personal-data breach, but a breach involving personal data requires prompt assessment.

Record:
- what data was affected
- confidentiality/integrity/availability impact
- number/type of affected users where known
- likely consequences
- containment/mitigation
- decision whether notification to authorities/users is required

If GDPR reporting may apply, obtain competent legal/privacy guidance promptly; statutory deadlines can be short.

## 9. Post-incident review

After recovery:
- write a timeline
- identify root cause
- identify why controls did/did not work
- create concrete remediation items
- update threat model, checklists and tests
- verify fixes in staging

# Data inventory and minimization policy

## 1. Authoritative rule

For the current NextUnlock design, the only Steam user datum intentionally persisted long-term is:

```text
SteamID64
```

This is a deliberate privacy and security decision.

## 2. Steam user data: persistent

| Data | Persist? | Reason |
|---|---:|---|
| SteamID64 | Yes | Stable external identity for Steam OpenID users |

Store it once on the user record and link all application-owned records through an internal `user_id`.

## 3. Steam user data: not persistent by default

Do not persist long-term:

- Steam password
- Steam Guard codes
- Steam login/session tokens
- end-user Steam Web API keys
- Steam e-mail address
- persona/display name
- avatar as user profile data
- owned-games/library snapshot
- playtime history
- player achievement state/history
- raw Steam API responses containing user-specific state
- historical synchronization snapshots of the user's Steam state

User-specific Steam data may be fetched on demand and temporarily cached with a documented TTL when needed for performance. Cache entries are not historical records and should expire.

## 4. Global Steam/game data

Non-user-specific data may be stored centrally when useful, for example:

- Steam AppID
- game name/metadata
- achievement API name
- achievement display name/description
- achievement icon references
- global achievement rarity
- schema/version/hash information for global game definitions

Do not duplicate global data per user.

## 5. NextUnlock-owned user data

NextUnlock-created data may be persistent because it is necessary for the product:

- internal user ID
- roadmaps
- roadmap steps
- manual roadmap checkmarks/progress
- settings/preferences required by the application
- session records
- timestamps required for operation/security

Example relationship:

```text
SteamID64
   -> internal user_id
      -> roadmaps
      -> roadmap_steps
      -> settings
      -> sessions
```

Although these are not raw Steam account data, they can still be personal data when linked to a user. Treat them accordingly.

## 6. Roadmap refresh

To detect progress since a roadmap was created, prefer comparing the saved NextUnlock roadmap/achievement references with current Steam data fetched on demand.

Do not create a broad permanent Steam achievement history merely for convenience.

If a future roadmap feature requires a minimal snapshot, document exactly which fields are required, why they are required, retention, and deletion behavior before adding them.

## 7. Historical/lost-perfect features

The current data-minimization decision intentionally does not maintain a permanent historical copy of a user's Steam library or achievement state.

Therefore, a future feature that requires historical Steam-derived user state (for example long-term "lost perfect" history independent of a saved roadmap) must not silently add broad persistence. Before implementation it requires an explicit architecture/privacy review and a minimal-data design.

## 8. Retention/deletion

Before public multi-user launch, define:
- account deletion
- deletion of NextUnlock-owned user data
- session cleanup
- cache expiry
- backup retention and deletion behavior
- legal/privacy documentation

The default should be: if a datum is no longer needed, do not keep it indefinitely.

---
title: "Concept: Identity Flow — Entra ID to Squid"
tags: [identity, entra-id, zitadel, netbird, squid, identity-bridge, zero-trust]
---

# Concept: Identity Flow — Entra ID to Squid

**One-line definition:** The complete identity propagation chain that carries a user's Entra ID group membership from Microsoft's cloud directory through four intermediary systems to the Squid proxy's per-request access decisions.

## How it applies here

In a commercial SASE product (Zscaler, Netskope), the agent sends identity headers with every request — identity propagation is a built-in feature of the client-to-cloud tunnel. In this PoC, no single component spans the full path from authentication to per-request filtering. Instead, identity propagation happens through a multi-hop chain where each system passes group membership to the next:

```
Entra ID (aplab.be)
  | OIDC token with groups claim (GUIDs) + cloud_displayname optional claim
  v
Zitadel (mgmt01 Docker)
  | Action 1: GUID -> clean name (strips 2ITcsc1A- prefix)
  | Action 2: setClaim('groups', [...]) into JWT
  v
NetBird Management (mgmt01 Docker)
  | JWT group sync: reads groups claim, creates/assigns auto_groups
  | Persona groups: Studenten, Docenten, Admins
  v
Identity Bridge (mgmt01 Docker, FastAPI)
  | Polls NetBird API every 30s: GET /api/peers + GET /api/users
  | Builds in-memory cache: {overlay_ip -> {email, groups[], os}}
  v
Squid (pop01, external_acl)
  | For each request: helper queries Identity Bridge with %SRC (overlay IP)
  | Returns persona group -> ACL evaluates per-group policies
  v
Per-request identity-based filtering
  (e.g., Studenten blocked from ChatGPT; Docenten allowed)
```

Each hop transforms the identity representation: GUIDs become clean names, clean names become JWT claims, JWT claims become auto-groups, auto-groups become cached IP-to-group mappings, and cached mappings become ACL decisions. A failure at any hop breaks the chain — but the fail-open design ensures that broken identity never blocks authentication, only restricts access to the most restrictive default policy.

## GroupSync Path B

The Entra ID `groups` claim contains GUIDs that are meaningless to NetBird. Two paths exist for converting these to usable group names:

**Path A — IdP Sync via API:** NetBird polls Microsoft Graph API directly to resolve group memberships. This requires a commercial NetBird license and is not available in the Community Edition used in this PoC.

**Path B — JWT group sync (implemented):** Zitadel Action 1 reads the `cloud_displayname` optional claim from the Entra ID token. This claim carries the human-readable group name with the tenant team prefix (`2ITcsc1A-Studenten`). The action strips the prefix, producing clean names (`Studenten`, `Docenten`, `Admins`). Action 2 then injects these clean names into the JWT issued to NetBird.

Path B has a fundamental limitation: group membership only updates when the user logs in. There is no background sync. If a user is added to a new Entra ID group, the change is not visible in the stack until that user's next OIDC authentication event.

See [Decision: GroupSync Path B](../decisions/groupsync-pad-b.md).

## Propagation delay

Changes in Entra ID group membership take time to propagate through the full chain. Each hop introduces its own delay:

| Hop | Delay | Trigger |
|-----|-------|---------|
| Entra ID -> Zitadel | Next user login | No background sync (Path B limitation) |
| Zitadel -> NetBird | Same login event | JWT group sync is synchronous |
| NetBird -> Identity Bridge | Up to 30s | Polling interval |
| Identity Bridge -> Squid | Next request | external_acl TTL (30s cache, 10s negative) |

**Total worst case:** next login + 30s + next request. In practice, this means a group membership change requires the user to re-authenticate with NetBird before the new group takes effect in Squid's filtering policies. For security-critical group removals (e.g., revoking admin access), this delay is significant.

## Fail-open design

The identity chain is designed to fail open at every hop rather than blocking access:

- **Zitadel Actions:** `allowed-to-fail: true` — authentication succeeds without group claims
- **Identity Bridge:** returns "unknown" for unresolvable IPs — Squid applies default restrictive policy
- **Squid external_acl:** falls through to non-identity URL filtering when Identity Bridge is unreachable

This is deliberate: Gate 3 (SWG pipeline — content inspection, malware scanning, DLP) operates independently of identity. A user without group claims still gets full content inspection; they simply cannot access group-specific policies (e.g., Docenten-only URL allowances). Security is layered, not dependent on a single identity chain.

## Where it appears in the stack

- **[Zitadel](../components/zitadel.md)** — OIDC broker, Action 1 (GUID-to-name mapping) and Action 2 (JWT group injection)
- **[NetBird](../components/netbird.md)** — JWT group sync reads the `groups` claim and creates auto_groups per persona
- **[Identity Bridge](../components/identity-bridge.md)** — polls NetBird API, builds overlay IP-to-group cache, serves Squid lookups
- **[Squid](../components/squid.md)** — external_acl queries Identity Bridge per request, enforces persona-based filtering policies
- **[Entra ID CA](../decisions/ca-posture-hybrid.md)** — Gate 1 Conditional Access policies evaluated during the OIDC flow

## Key distinctions

**Identity vs authentication:** Authentication (proving who you are) happens once during the OIDC login flow through Zitadel and Entra ID. Identity propagation (carrying that proof through the stack so downstream systems can make access decisions) happens continuously through the polling and caching chain. A user can be authenticated without having their identity propagated — this is the fail-open state.

**Path A vs Path B:** Path A (IdP Sync) polls Graph API directly and updates group membership in the background — it works even when users are not actively logging in. Path B (JWT group sync) relies on the OIDC token issued at login time — group changes only propagate when the user re-authenticates. This PoC uses Path B because it is available in NetBird Community Edition.

**Fail-open vs fail-closed:** The identity chain fails open by design. This contrasts with the content inspection chain (Gate 3), where ClamAV scanning is not bypassed when the service is down. The rationale: identity controls determine *which* policies apply, while content inspection provides baseline security for *all* traffic regardless of identity.

## Sources

- `raw/18_mei_SASE_PoC_Addendum_H_Identity_Bridge_v1.md` (Identity Bridge architecture)
- `raw/18_mei_SASE_PoC_Addendum_GroupSync.md` (JWT group sync, Path B)
- `raw/GroupSync_Correcties_Sessie1(1).md` (Zitadel Actions implementation)

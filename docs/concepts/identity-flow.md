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
  | OIDC token: groups claim emits display names (2ITCSC1A-Studenten) via cloud_displayname
  v
Zitadel (mgmt01 Docker)
  | Action 1: allowlist-maps display name -> clean name (2ITCSC1A-Studenten -> Studenten)
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

Each hop transforms the identity representation: prefixed display names become clean names, clean names become JWT claims, JWT claims become auto-groups, auto-groups become cached IP-to-group mappings, and cached mappings become ACL decisions. A failure at any hop breaks the chain — but the fail-open design ensures that broken identity never blocks authentication, only restricts access to the most restrictive default policy.

## GroupSync: sync mechanism and Path B

**Sync mechanism — JWT group sync.** NetBird Community Edition cannot use the IdP Sync (Graph API polling) or SCIM provisioning mechanisms — both require a NetBird Cloud or Commercial License. The only mechanism available in CE is **JWT group sync**: NetBird reads the `groups` claim from the ID token on each user login and creates/assigns auto-groups accordingly. Its inherent limitation is that group membership only updates at login — there is no background sync. If a user is added to a new Entra ID group, the change is not visible in the stack until that user's next OIDC authentication event.

**Prefix handling — Path B (implemented).** Entra ID group names carry a mandatory `2ITCSC1A-` team prefix (e.g. `2ITCSC1A-Studenten`). Because `cloud_displayname` is attached to the groups claim (see [Runbook 08](../runbooks/08-groupsync.md)), the claim emits each group's *display-name string* rather than its ObjectID GUID. Two paths were considered:

- **Path A — carry the prefix everywhere:** the full name `2ITCSC1A-Studenten` propagates unchanged to NetBird, the Identity Bridge ACL, and Addendum J's `NETBIRD_POLICY_GROUPS`. Traceable but verbose, and the prefix cascades through every downstream config.
- **Path B — Zitadel strips the prefix (implemented):** Action 1 allowlist-maps the name string `2ITCSC1A-Studenten` → `Studenten` before NetBird sees it; Action 2 injects the clean names (`Studenten`, `Docenten`, `Admins`) into the JWT. Internal configs stay readable and Addendum J's existing `Studenten,Docenten,Admins` is already correct.

Path B's hard precondition is that `cloud_displayname` works — a GUID is not in the allowlist and cannot resolve to `Studenten`. That precondition was verified, and Path B is confirmed working for all three personas.

See [Decision: GroupSync Path B](../decisions/groupsync-pad-b.md).

## Propagation delay

Changes in Entra ID group membership take time to propagate through the full chain. Each hop introduces its own delay:

| Hop | Delay | Trigger |
|-----|-------|---------|
| Entra ID -> Zitadel | Next user login | No background sync (JWT group sync limitation) |
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

- **[Zitadel](../components/zitadel.md)** — OIDC broker, Action 1 (allowlist-maps the group display-name string to the clean name) and Action 2 (JWT group injection)
- **[NetBird](../components/netbird.md)** — JWT group sync reads the `groups` claim and creates auto_groups per persona
- **[Identity Bridge](../components/identity-bridge.md)** — polls NetBird API, builds overlay IP-to-group cache, serves Squid lookups
- **[Squid](../components/squid.md)** — external_acl queries Identity Bridge per request, enforces persona-based filtering policies
- **[Entra ID CA](../decisions/ca-posture-hybrid.md)** — Gate 1 Conditional Access policies evaluated during the OIDC flow

## Key distinctions

**Identity vs authentication:** Authentication (proving who you are) happens once during the OIDC login flow through Zitadel and Entra ID. Identity propagation (carrying that proof through the stack so downstream systems can make access decisions) happens continuously through the polling and caching chain. A user can be authenticated without having their identity propagated — this is the fail-open state.

**JWT group sync vs IdP Sync:** IdP Sync (and SCIM) poll or push group membership in the background — they update even when users are not actively logging in — but both require a NetBird Cloud or Commercial License, so neither is available in the Community Edition used here. JWT group sync, the only CE-available mechanism, relies on the OIDC token issued at login time, so group changes only propagate when the user re-authenticates. Separately, **Path A vs Path B** refers to prefix handling *within* JWT group sync (Path B strips the `2ITCSC1A-` prefix — see above), not to the choice of sync mechanism.

**Fail-open vs fail-closed:** The identity chain fails open by design. This contrasts with the content inspection chain (Gate 3), where ClamAV scanning is not bypassed when the service is down. The rationale: identity controls determine *which* policies apply, while content inspection provides baseline security for *all* traffic regardless of identity.

## Sources

- `18_mei_SASE_PoC_Addendum_H_Identity_Bridge_v1.md` (Identity Bridge architecture)
- `18_mei_SASE_PoC_Addendum_GroupSync.md` (JWT group sync, Path B)
- `GroupSync_Correcties_Sessie1(1).md` (Zitadel Actions implementation)

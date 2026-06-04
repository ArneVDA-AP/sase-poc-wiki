---
title: "Zitadel — OIDC Identity Provider Broker"
tags: [zitadel, oidc, identity, entra-id, netbird, ztna]
---

# Zitadel — OIDC Identity Provider Broker

**Role:** OIDC Identity Provider broker — sits between NetBird and Entra ID, translating Microsoft identities into JWT claims that NetBird can consume for group-based access control.  
**Version:** Deployed via NetBird quickstart Docker Compose on mgmt01  
**Config location:** Docker Compose stack on mgmt01 (part of NetBird deployment)

---

## How it works in this stack

Zitadel is the OIDC IdP that NetBird authenticates against. It was installed as part of the NetBird quickstart script, which makes Zitadel the primary OIDC issuer rather than connecting NetBird directly to an external IdP. Entra ID (aplab.be tenant) is configured as an external IdP *within Zitadel*, so the full authentication flow is:

```
NetBird client
  → Zitadel (OIDC issuer on mgmt01)
    → Entra ID (aplab.be tenant, external IdP)
      → Microsoft login page
    ← OIDC token: groups claim = display-name strings (via cloud_displayname)
  ← JWT with clean groups claim
→ NetBird Management reads groups from JWT
```

Zitadel adds a central user-management layer with roles and groups independent of Entra ID configuration. This is architecturally significant: it decouples the ZTNA overlay's identity model from Microsoft's directory, allowing group naming and mapping to be controlled locally.

---

## Zitadel Actions

Two custom Actions form the critical bridge between Entra ID's raw claims and the clean group names NetBird requires. Both are JavaScript snippets executed server-side by Zitadel during the authentication flow.

### Action 1 — External Authentication (Pre-creation hook)

When a user logs in via Entra ID for the first time, the Entra ID token's `groups` claim carries the user's group memberships. Because `cloud_displayname` is attached to the `groups` claim in the app manifest (see [Runbook 08](../runbooks/08-groupsync.md)), the claim emits each group's **display-name string** (e.g. `2ITCSC1A-Studenten`) rather than its ObjectID GUID (e.g. `e5f8a2b3-...`).

Action 1 (`mapEntraGroupsToMetadata`) resolves the clean internal names with an **allowlist** keyed on those display-name strings:

1. Read the `groups` claim (display-name strings)
2. Look each name up in a hardcoded allowlist — `2ITCSC1A-Studenten` → `Studenten`, `2ITCSC1A-Docenten` → `Docenten`, `2ITCSC1A-Admins` → `Admins` (this is [GroupSync Path B](../decisions/groupsync-pad-b.md))
3. Write the matched clean names to the user metadata key `sase_groups` (comma-joined)

The lookup is keyed on the **name string**, which is why `cloud_displayname` is a hard precondition — a raw GUID cannot be allowlisted to `Studenten`. The allowlist is deliberately **fail-closed**: a group not in the map is silently dropped rather than passed through (stricter than a blind prefix-strip). The clean names keep downstream NetBird ACLs and Squid policies readable.

### Action 2 — Complement Token

Adds the `groups` claim to the JWT token that Zitadel issues to NetBird. Without this action, the JWT contains authentication information but no group membership, and NetBird never sees which persona group the user belongs to.

The action reads `sase_groups` from the user's metadata and calls `setClaim('groups', [...])` to inject those clean names into the outgoing JWT. NetBird's JWT group sync then reads this claim and creates or assigns auto-groups accordingly.

### Fail-open design

Both actions have `allowed-to-fail: true`. If either action fails, authentication still succeeds — the user gets a valid JWT but without group claims. This is a deliberate design choice: fail-open at the auth layer is acceptable because Gate 3 (the SWG pipeline on pop01) still enforces content inspection regardless of group membership. A user without group claims receives the most restrictive default policy rather than being locked out entirely.

---

## Configuration

### App Registration (Entra ID side)

The Entra ID app registration that Zitadel federates to is sandbox-specific:

| Property | Value |
|----------|-------|
| App name | `2ITCSC1A-Netbird-Sandbox` |
| Client ID | `11803ee8-eb15-462c-a286-5415c17a29c6` |
| Tenant ID | `23e9bcdc-5cb9-4867-9310-76cc0b462ddc` |

This replaced the shared registration `cebe0d74-be9f-49ac-9f35-65f11586c1bb` which was used by other teams. The sandbox-specific registration avoids cross-team interference with Conditional Access policies and token configuration.

### Token Configuration (Entra ID side)

The `cloud_displayname` property must be attached to the `groups` claim in the Entra ID app registration manifest — it is not selectable in the Token Configuration UI and must be added by editing the manifest directly (see [Runbook 08](../runbooks/08-groupsync.md)). Without it, the `groups` claim falls back to GUIDs, Action 1's allowlist has no name string to match (a GUID is never in the allowlist), and group resolution fails silently (due to `allowed-to-fail: true`).

---

## Integration points

| Component | Direction | What |
|-----------|-----------|------|
| [NetBird](netbird.md) | ← OIDC | Zitadel is NetBird's configured OIDC issuer; JWT group sync depends on Action 2 |
| Entra ID (aplab.be) | → federation | External IdP providing Microsoft identities and group membership |
| [Identity Bridge](identity-bridge.md) | indirect | Groups propagated through Zitadel → NetBird → Identity Bridge polling chain |
| [Squid](squid.md) | indirect | End consumer of identity: persona groups determine per-request filtering policy |

---

## Known issues / gotchas

**No stdout/console logging in Actions** — Zitadel Actions have no debugging output mechanism. There is no way to log intermediate values or trace execution within an action script. The only verification method is examining JWT claims in NetBird after a successful login, making iterative development slow and error-prone.

**GitHub #5399 (embedded-Dex scope drop) — does not apply to this stack** — Upstream #5399 reports that on NetBird builds using *embedded Dex* + Zitadel, JWT group sync fails out-of-the-box because `AUTH_SUPPORTED_SCOPES` is missing the groups scope. This sandbox runs the **older multi-container NetBird stack with no Dex container** (Management v0.67.0 validates Zitadel's token directly — Verslag30), so the action-injected `groups` claim flows through without any scope fix. The scope-drop variant of #5399 is therefore not applicable here; no Docker Compose patching is needed for group sync. (The separate "filled JWT allow-groups → 401 lockout" hazard, sometimes also tracked under #5399, still applies — see below and [NetBird](netbird.md).)

**`cloud_displayname` dependency** — If `cloud_displayname` is not attached to the `groups` claim in the app manifest, the claim carries GUIDs instead of names and Action 1's allowlist matches nothing, silently dropping every group. The user authenticates successfully but without group resolution. Because `allowed-to-fail: true` suppresses the error, this failure mode is invisible without checking JWT claims downstream.

**Action execution is opaque** — There is no dashboard view showing action execution history or failure counts. Debugging requires end-to-end testing: log in as a test user, then inspect the resulting JWT in NetBird to verify group claims are present and correctly mapped.

---

## Related

- [NetBird](netbird.md)
- [Identity Bridge](identity-bridge.md)
- [Concept: Identity Flow](../concepts/identity-flow.md)
- [Decision: Zitadel as IdP broker](../decisions/zitadel-idp-broker.md)
- [Decision: GroupSync Path B](../decisions/groupsync-pad-b.md)
- [Runbook: ZTNA Overlay](../runbooks/02-ztna-overlay.md)

---
title: "Decision: GroupSync Path B — Zitadel Strips Entra ID Prefix"
tags: [decision, groupsync, zitadel, entra-id, netbird]
---

# Decision: GroupSync Path B — Zitadel Strips Entra ID Prefix

**Status:** Implemented  
**Date:** May 2026 (Verslag30–34; Path B confirmed for all three personas in Verslag34)

## Context

Entra ID group names carry a mandatory `2ITCSC1A-` prefix imposed by a lector mandate (the tenant is shared across multiple student projects). For example, the group for students appears as `2ITCSC1A-Studenten` in Entra ID. This prefix must be handled somewhere in the synchronization pipeline because NetBird access policies and Squid ACLs operate on group names — policies referencing `2ITCSC1A-Studenten` instead of `Studenten` are harder to read, harder to maintain, and propagate the prefix through every downstream component (Identity Bridge, NATS subjects, Squid external ACL responses).

## Options considered

| Option | Pro | Con |
|--------|-----|-----|
| **Path A: Carry prefix everywhere** | No transformation logic needed; names are 1:1 with Entra ID | Every policy, ACL, and NATS subject must use the full prefixed name (`2ITCSC1A-Studenten`). Reduces readability. Prefix leaks into Identity Bridge mapping, NATS subject hierarchy, and Squid ACL output |
| **Path B: Zitadel strips prefix** | Clean internal names (`Studenten`). Policies and ACLs are readable. No prefix propagation through the stack | Requires the existing Zitadel Action to allowlist-map the group display-name string to the clean name. Hard precondition: `cloud_displayname` must work, so the JWT claim carries the name `2ITCSC1A-Studenten` rather than an ObjectID GUID (a GUID is not in the allowlist and cannot be mapped to `Studenten`). A mismatch between the mapped name and what NetBird/Squid expect causes a lockout |

## Decision

Path B — the Zitadel Action already in the OIDC chain maps the group's display-name string to the clean internal name (`2ITCSC1A-Studenten` → `Studenten`) via a hardcoded **allowlist**, dropping any group not in the allowlist (fail-closed). The net effect is that the `2ITCSC1A-` prefix never reaches NetBird. This depends on `cloud_displayname` being enabled in the Entra app manifest so the `groups` claim emits the display name rather than the GUID; that precondition was verified, and Path B was confirmed working for all three personas (Studenten and Admins by Verslag31, Docenten by Verslag34). The prefix exists only in Entra ID; all downstream components (Zitadel claims, NetBird JWT auto-groups, Identity Bridge, NATS subjects, Squid ACLs) use the clean name.

The primary reason is policy readability: when a NetBird ACL or Squid rule references `Studenten`, it is immediately clear what group is being controlled. With `2ITCSC1A-Studenten`, every reader must mentally strip the prefix to understand the policy intent. Path B also leaves Addendum J's `NETBIRD_POLICY_GROUPS` config (`Studenten,Docenten,Admins`) already correct and reuses an existing transformation point rather than adding one.

## Consequences

- Path B's correctness hinges on `cloud_displayname`: if it ever stops emitting display names (e.g., the groups are treated as on-prem-synced), the claim falls back to GUIDs and the allowlist (keyed on the name string) matches nothing — every group is dropped fail-closed, and the fragile fallback would be GUID matching in ACLs or a GUID→name lookup. This is verified in the GroupSync `cloud_displayname` step before relying on Path B.
- The clean internal name must match **byte-for-byte** across the Identity Bridge external ACL, Addendum J's `NETBIRD_POLICY_GROUPS`, and every NetBird ACL policy. A mismatch (case, spacing) blocks all affected peers with 401 on every API request — including the admin — and recovery requires direct SQLite intervention on mgmt01 (GitHub #5399).
- The Identity Bridge does not need to perform any prefix stripping; it sees the already-clean group name (`Studenten`) on the peer when it polls the NetBird API.
- If the lector mandate changes the prefix convention, only the Zitadel Action's strip logic needs to be updated — no downstream changes required.

See also: [Component: Zitadel](../components/zitadel.md), [Component: Identity Bridge](../components/identity-bridge.md), [Component: NetBird](../components/netbird.md), [Concept: Identity Flow](../concepts/identity-flow.md)

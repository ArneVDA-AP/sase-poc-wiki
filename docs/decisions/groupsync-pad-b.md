---
title: "Decision: GroupSync Path B — Zitadel Strips Entra ID Prefix"
tags: [decision, groupsync, zitadel, entra-id, netbird]
---

# Decision: GroupSync Path B — Zitadel Strips Entra ID Prefix

**Status:** Implemented  
**Date:** May 2026 (Verslag30)

## Context

Entra ID group names carry a mandatory `2ITcsc1A-` prefix imposed by a lector mandate (the tenant is shared across multiple student projects). For example, the group for students appears as `2ITcsc1A-Studenten` in Entra ID. This prefix must be handled somewhere in the synchronization pipeline because NetBird access policies and Squid ACLs operate on group names — policies referencing `2ITcsc1A-Studenten` instead of `Studenten` are harder to read, harder to maintain, and propagate the prefix through every downstream component (Identity Bridge, NATS subjects, Squid external ACL responses).

## Options considered

| Option | Pro | Con |
|--------|-----|-----|
| **Path A: Carry prefix everywhere** | No transformation logic needed; names are 1:1 with Entra ID | Every policy, ACL, and NATS subject must use the full prefixed name (`2ITcsc1A-Studenten`). Reduces readability. Prefix leaks into Identity Bridge mapping, NATS subject hierarchy, and Squid ACL output |
| **Path B: Zitadel strips prefix** | Clean internal names (`Studenten`). Policies and ACLs are readable. No prefix propagation through the stack | Requires a Zitadel Action to map Entra ID GUID to clean name. Mismatch between Zitadel output and NetBird JWT sync could cause lockout |

## Decision

Path B — a Zitadel Action maps the Entra ID group GUID to a clean internal name, stripping the `2ITcsc1A-` prefix. The prefix exists only in Entra ID; all downstream components (Zitadel claims, NetBird JWT auto-groups, Identity Bridge, NATS subjects, Squid ACLs) use the clean name.

The primary reason is policy readability: when a NetBird ACL or Squid rule references `Studenten`, it is immediately clear what group is being controlled. With `2ITcsc1A-Studenten`, every reader must mentally strip the prefix to understand the policy intent.

## Consequences

- The Zitadel Action must maintain an allowlist mapping from Entra ID group GUIDs to clean names. Adding a new group in Entra ID requires updating this mapping.
- A mismatch between Zitadel's output group name and what NetBird expects in JWT claims will result in peers not being assigned to the correct auto-groups — effectively a lockout scenario.
- The Identity Bridge does not need to perform any prefix stripping; it receives already-clean names from the Zitadel-issued JWT.
- NATS subjects use clean group names (e.g., `groupsync.Studenten`), keeping the subject hierarchy readable.
- If the lector mandate changes the prefix convention, only the Zitadel Action mapping needs to be updated — no downstream changes required.

See also: [Component: Zitadel](../components/zitadel.md), [Component: Identity Bridge](../components/identity-bridge.md), [Component: NetBird](../components/netbird.md), [Concept: Identity Flow](../concepts/identity-flow.md)

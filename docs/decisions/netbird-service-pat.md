---
title: "Decision: NetBird Service-User PAT for Identity Bridge"
tags: [decision, netbird, identity-bridge, api]
---

# Decision: NetBird Service-User PAT for Identity Bridge

**Status:** Implemented  
**Date:** May 2026 (Verslag31)

## Context

Identity Bridge must poll the NetBird Management API to synchronize group membership from Zitadel into NetBird peer groups. The obvious approach is to use a Personal Access Token (PAT) from an existing admin user. However, NetBird issue #3127 reveals a destructive side effect: when the API is polled using a regular user's PAT, the system strips all JWT-propagated auto-groups from every peer belonging to that user on each API call. This effectively removes group-based policy assignments for the polling user's peers.

## Options considered

| Option | Pro | Con |
|--------|-----|-----|
| **Regular user PAT** | Simple to set up; reuse existing admin account | Triggers issue #3127 — every API poll strips JWT-propagated auto-groups from all peers of the PAT owner. Breaks group-based access policies silently |
| **Service-user PAT with admin role** | Service user has no JWT-propagated auto-groups and no peers — the strip-effect has nothing to act on | Requires creating a dedicated service user with admin role in NetBird; PAT must be securely stored |

## Decision

A dedicated service user named `identity-bridge` with admin role is created in NetBird. This service user's PAT is used for all API polling by Identity Bridge.

The key insight: issue #3127 strips JWT-propagated auto-groups from all peers of the PAT owner. A service user has no JWT-propagated auto-groups (it never authenticates via OIDC) and has no peers (it is not a device). Therefore, the strip-effect has nothing to act on — it fires on an empty set.

## Consequences

- A service user with admin role must exist in NetBird management. This is a privileged account used solely for API access.
- The PAT must be stored securely in the `.env` file of the Identity Bridge deployment. It must not be committed to version control.
- If the service user is accidentally assigned peers or groups, issue #3127 could re-emerge. The service user must remain peer-free and group-free.
- The workaround is specific to the current NetBird version — if #3127 is fixed upstream, a regular user PAT would become safe again, but the service-user approach remains cleaner regardless.

See also: [Finding: NetBird issue #3127](../findings/netbird-issue-3127.md), [Component: NetBird](../components/netbird.md), [Component: Identity Bridge](../components/identity-bridge.md)

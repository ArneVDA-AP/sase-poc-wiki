---
title: "Finding: NetBird API polling with user PAT strips JWT auto-groups"
tags: [finding, netbird, identity-bridge, api]
---

# Finding: NetBird API polling with user PAT strips JWT auto-groups

**Component:** [Identity Bridge](../components/identity-bridge.md), [NetBird](../components/netbird.md)  
**Severity:** Blocker

## What happened

Polling the NetBird Management API with a regular user PAT stripped JWT-propagated auto-groups from ALL peers belonging to that user at every API call. Groups disappeared from peers within seconds of starting the Identity Bridge. Peer routing and ACL policies depending on those groups broke immediately.

## Root cause

NetBird interprets an API call without JWT group context as "no groups present anymore" and strips propagated groups from the calling user's peers. A PAT carries only the API authentication identity, not the original JWT claims — so the group context is absent. The management server reconciles the user's group membership on every authenticated API interaction, and an interaction without JWT groups results in group removal.

## Resolution / workaround

Use a **service-user** with an admin-role PAT for all Identity Bridge API polling. Service users have no JWT groups and no peers of their own — the strip-effect cannot occur because there are no propagated groups to reconcile and no peers to affect.

```
# NetBird Dashboard → Users → Service Users → Create
# Role: admin
# Generate PAT → use in Identity Bridge config
```

## Lessons

- Admin-role service accounts are the correct pattern for API polling in NetBird CE self-hosted when JWT group sync is active
- Regular user PATs are unsuitable for automated API access in environments with JWT group propagation — the side-effect is destructive and silent
- Related to NetBird issue #3127

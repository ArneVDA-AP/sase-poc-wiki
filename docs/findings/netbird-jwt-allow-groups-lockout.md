---
title: "Finding: NetBird JWT allow groups field causes total lockout if misconfigured"
tags: [finding, netbird, jwt, authentication]
---

# Finding: NetBird JWT allow groups field causes total lockout if misconfigured

**Component:** [NetBird](../components/netbird.md)  
**Severity:** Blocker

## What happened

The "JWT allow groups" field in NetBird Dashboard (Settings -> Groups) was filled with a group name that did not match any JWT claim value. The result was immediate and total: ALL users received 401 Unauthorized. The entire NetBird deployment became inaccessible — no user could authenticate, no peer could connect, and the dashboard itself became unusable for correcting the mistake.

## Root cause

NetBird checks the JWT `groups` claim against the allow list on every authentication attempt. The field works as a **whitelist**, not a filter: if the allow list is non-empty and no group in the user's JWT matches any entry in the list, authentication is rejected. There is no "soft fail" or grace period — the lockout is immediate and applies to all users, including administrators.

## Resolution / workaround

**Prevention:** Leave the field **empty** (= no filtering, all JWT groups are synchronized). Only fill it if you are certain the values match exactly what appears in the JWT `groups` claim.

**Recovery from lockout:** Direct SQLite access on the management container is required:

```bash
# Access the NetBird management container
docker exec -it netbird-management sh

# Find and clear the JWT allow groups setting
sqlite3 /var/lib/netbird/store.db
# Identify the relevant table/column and clear the allow groups value
```

## Lessons

- Test JWT group sync with the allow field empty first — only populate it after confirming the exact claim values
- The field name "JWT allow groups" is misleading — it functions as a strict whitelist, not a permissive filter
- Recovery requires database-level access, which may not be readily available in all deployment scenarios — always have a backup plan before modifying this setting
- Reference: NetBird issue #5399, v0.62 documentation

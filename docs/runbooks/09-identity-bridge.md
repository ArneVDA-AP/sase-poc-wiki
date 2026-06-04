---
title: "Runbook: Identity Bridge"
tags: [runbook, identity-bridge, fastapi, squid, docker]
---

# Runbook: Identity Bridge

**Node(s):** mgmt01 (Docker — Identity Bridge container), pop01 (Squid external ACL)
**Prerequisites:** [Runbook 02: ZTNA Overlay](02-ztna-overlay.md) completed (NetBird operational), [Runbook 08: GroupSync](08-groupsync.md) completed
**Status:** Operational

---

## Prerequisites checklist

- [ ] NetBird overlay operational ([Runbook 02](02-ztna-overlay.md))
- [ ] GroupSync complete — persona groups (Studenten/Docenten/Admins) populated in NetBird ([Runbook 08](08-groupsync.md))
- [ ] Docker installed on mgmt01
- [ ] NetBird service user PAT available for API authentication

---

## Step 1: Deploy Identity Bridge container on mgmt01

The Identity Bridge is a FastAPI application that maps NetBird peer overlay IPs to persona groups by polling the NetBird Management API.

Build and run the Docker container on mgmt01:

1. Place the Identity Bridge source code on mgmt01
2. Build the Docker image
3. Configure environment variables:

| Variable | Value | Purpose |
|----------|-------|---------|
| `NETBIRD_API_URL` | NetBird API endpoint (localhost or Docker network) | API base URL |
| `NETBIRD_PAT` | Service user Personal Access Token | API authentication |
| `POLL_INTERVAL` | `30` (seconds, default) | How often to refresh peer-to-group mappings |

4. Start the container, exposing the lookup port on `192.168.122.23`

> **Gotcha: Use a service user PAT, not a personal admin token.** The PAT must have read access to peers and groups. See [Decision: NetBird Service PAT](../decisions/netbird-service-pat.md) for why a dedicated service user was created.

---

## Step 2: Verify Identity Bridge is running

```bash
# Check container is up
docker ps | grep identity-bridge

# Test the lookup endpoint directly (the X-Bridge-Secret header is required — a bare
# request returns HTTP 401, fail-secure)
curl -H "X-Bridge-Secret: <secret>" "http://192.168.122.23:<port>/lookup?ip=<known-peer-ip>"
# Expected: JSON with the peer's full group membership (status/user/groups/os)
```

The Identity Bridge polls the NetBird API every 30 seconds and caches the IP-to-group mapping in memory. The first response may take up to 30 seconds after startup while the initial poll completes.

---

## Step 3: Configure Squid external_acl_type on pop01

On pop01, configure Squid to use the Identity Bridge as an external ACL helper for identity-based filtering.

Edit the Squid configuration to add:

1. **External ACL type definition:** a helper script that queries the Identity Bridge at `http://192.168.122.23:<port>/lookup?ip=%SRC`
2. **ACL definitions** based on persona groups:

| ACL name | Persona group | Access level |
|----------|---------------|--------------|
| `studenten_acl` | Studenten | Restricted (blocked categories) |
| `docenten_acl` | Docenten | Standard (most sites allowed) |
| `admins_acl` | Admins | Full access |

3. **http_access rules** that use these ACLs to enforce differentiated filtering

Reload Squid after configuration:

```bash
squid -k reconfigure
```

---

## Step 4: Verify identity appears in Squid logs

1. Browse from mobile01 (as a test user with a known persona)
2. Check Squid access log on pop01:

```bash
tail -f /var/log/squid/access.log
```

**Expected:** Log entries include the identity group (Studenten/Docenten/Admins) for each request, proving that Squid successfully queries the Identity Bridge for every connection.

---

## Step 5: Test identity-based filtering

Test that policy enforcement differs by persona:

**Student (restricted):**

```bash
curl -x http://100.70.154.79:3128 https://chatgpt.com
# Expected: 403 Forbidden (blocked for students)
```

**Teacher (standard):**

```bash
curl -x http://100.70.154.79:3128 https://chatgpt.com
# Expected: 200 OK (allowed for teachers)
```

> **Note:** To test different personas, you need to log in as users who belong to different Entra ID security groups (`2ITCSC1A-Studenten` vs `2ITCSC1A-Docenten` vs `2ITCSC1A-Admins`), which GroupSync maps to the internal NetBird persona groups Studenten/Docenten/Admins.

---

## Final verification

End-to-end validation:

1. `netbird up` as a student user → NetBird Dashboard shows Studenten group
2. Identity Bridge `/lookup` returns "Studenten" for that user's overlay IP
3. `curl -x http://100.70.154.79:3128 https://chatgpt.com` → 403
4. `netbird up` as a teacher user → NetBird Dashboard shows Docenten group
5. Identity Bridge `/lookup` returns "Docenten" for that user's overlay IP
6. `curl -x http://100.70.154.79:3128 https://chatgpt.com` → 200

---

## Checklist

- [ ] Identity Bridge Docker container running on mgmt01
- [ ] Service user PAT configured for NetBird API access
- [ ] Polling interval set (default 30s)
- [ ] `/lookup?ip=...` endpoint returns correct persona for known peers
- [ ] Squid external_acl_type configured to query Identity Bridge
- [ ] Squid ACLs defined for Studenten/Docenten/Admins
- [ ] Squid access.log shows identity group per request
- [ ] Student blocked from ChatGPT (403)
- [ ] Teacher allowed to ChatGPT (200)

---

## Related

- [Component: Identity Bridge](../components/identity-bridge.md)
- [Component: NetBird](../components/netbird.md)
- [Component: Squid](../components/squid.md)
- [Decision: NetBird Service PAT](../decisions/netbird-service-pat.md)
- [Decision: GroupSync PAD B](../decisions/groupsync-pad-b.md)
- [Concept: Zero Trust](../concepts/zero-trust.md)
- [Runbook 08: GroupSync](08-groupsync.md)

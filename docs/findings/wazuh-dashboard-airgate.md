---
title: "Finding: Wazuh Dashboard Offline redirect caused by empty manager UUID"
tags: [finding, wazuh, dashboard, air-gap, docker]
---

# Finding: Wazuh Dashboard Offline redirect caused by empty manager UUID

**Component:** [Wazuh](../components/wazuh.md)  
**Severity:** Gotcha (resolved 2 juni 2026)

## What happened

After a `docker compose down -v` (which wiped named Docker volumes) followed by a manager downgrade (4.14.5 → 4.14.3), the Wazuh Dashboard's app modules (Security Events, Agents tab) showed **"Status: Offline"** / **"Updates status: Error checking updates"** and hard-redirected to the API-configuration page on every load. Simultaneously, the pop01 agent (ID 001) disconnected. The Discover interface continued to work throughout.

## Root cause

Two distinct problems combined:

**1. Empty manager UUID (cause of the Offline redirect)**  
`docker compose down -v` wipes named Docker volumes, including `global.db`. On a fresh `global.db`, the manager UUID is empty (`"uuid": ""`). The dashboard's `POST /api/check-api` returns HTTP 500 — "Could not obtain manager UUID"; the dashboard interprets this as the manager being offline, sets "Status: Offline", and gates all app modules behind the API-configuration redirect.

**2. Version skew (cause of agent disconnect)**  
The OPNsense plugin `os-wazuh-agent` delivers a fixed agent version (4.14.5). Wazuh's compatibility rule requires **manager ≥ agent**. Downgrading the manager to 4.14.3 put it below the agent, causing enrollment refusal and the `(4101)` version-warning in agent logs.

**Important:** `GET /manager/version/check` → HTTP 500 ("Error checking updates") is caused by the air-gap blocking the CTI update endpoint, but it is a *separate* issue from the Offline redirect. In Wazuh 4.14.5 (fix #8130), this check is non-blocking — it does **not** cause the Offline redirect. The redirect is caused solely by the empty UUID.

## Two structural rules

**Rule 1 — The agent version sets the manager floor.**  
The OPNsense plugin pins the agent version (currently 4.14.5). Downgrading a FreeBSD package is brick-risk. Always keep manager ≥ agent. Check the agent version before any manager version change.

**Rule 2 — Version work = in-place tag bump, never `down -v`.**  
`docker compose down -v` destroys named volumes: UUID (`global.db`), agent registration (`client.keys`), and indexed alerts. For version changes: edit the image tag in `docker-compose.yml`, run `docker compose pull` then `docker compose up -d` (no `-v`). The UUID and agent registration survive.

## Resolution

Resolved 2 juni 2026 via in-place bump to Wazuh 4.14.5 GA (no `-v`), which preserved the `global.db` UUID from the 4.14.3 run. The pop01 agent re-enrolled and shows Active. Dashboard app loads without redirect.

All acceptance criteria met:
- `wazuh-control info` → `v4.14.5`
- `GET /manager/info` → `uuid` non-empty; dashboard app loads without redirect
- `agent_control -l` → `001 OPNsense.internal Active`
- Discover `rule.id:100540` → `data.producer:unbound` document indexed

## Lessons

- The "Status: Offline" redirect is caused by an **empty manager UUID**, not by DNS or the air-gap CTI check. Verify with `GET /manager/info` (Dev Tools in the dashboard) before chasing connectivity.
- `docker compose down -v` is destructive for Wazuh: UUID and agent registration are in named volumes, not bind-mounts. Always use `docker compose up -d` (no `-v`) for version work.
- The CTI-500 (`Error checking updates`) is cosmetic in Wazuh 4.14.5+ and can be ignored in air-gapped deployments.
- The OPNsense agent version pins the manager floor — never set the manager below the agent version.

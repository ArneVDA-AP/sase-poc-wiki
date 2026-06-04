---
title: "Finding: Wazuh Dashboard Apps section fails in air-gapped environment"
tags: [finding, wazuh, dashboard, air-gap]
---

# Finding: Wazuh Dashboard Apps section fails in air-gapped environment

**Component:** [Wazuh](../components/wazuh.md)  
**Severity:** Gotcha (known limitation)

## What happened

The Wazuh Dashboard's app modules (Security Events, the Agents tab) are gated: instead of loading, the dashboard reports the manager as **"Status: Offline"** with **"Updates status: Error checking updates"** and hard-redirects to the server-APIs (API configuration) page. The **Discover** interface, by contrast, works fully.

This is misleading, because the manager API is actually healthy — `authenticate`, `info`, and `stats` all return HTTP 200. The "Offline" verdict is a false negative produced by a single failing check, not by an unreachable manager.

## Root cause

Two internet-dependent checks fail in the air-gapped sandbox and cascade into the offline determination:

1. On the manager, `GET /manager/version/check` returns **HTTP 500**. This endpoint performs an internet-dependent update / CTI check; with no outbound internet in the sandbox it errors rather than returning a version.
2. The dashboard's `POST /api/check-api` then returns **HTTP 500 — "Could not obtain manager UUID"**. The dashboard treats this failure as the manager being offline, sets "Status: Offline" / "Updates status: Error checking updates", and gates the app modules behind the API-configuration redirect.

So the gate is **not** a DNS / hostname-resolution problem and not a perpetual hang — it is a fast 500 from an internet-dependent health check being interpreted as "manager offline".

## Resolution / workaround

No definitive fix was applied; the gate is accepted as a known air-gap limitation. Use **Discover** as the primary analysis interface — it queries the indexer directly and is unaffected by the manager's version/UUID check:

- Navigate to Discover in the Wazuh Dashboard sidebar
- Select the `wazuh-alerts-*` index pattern
- Use KQL or Lucene query syntax for filtering (e.g. `rule.groups:"sase"`)
- Create saved searches and visualizations as needed

Candidate fix paths (to revisit with Gerben, not yet validated): disable the update / CTI check on the manager side (its `api.yaml`) or the corresponding dashboard plugin setting so `check-api` no longer depends on the version check, then apply with `wazuh-control restart` (never `docker restart` — bind-mounts are overwritten in place). See [Component: Wazuh](../components/wazuh.md).

## Lessons

- An "Offline" verdict in the dashboard app is not proof the manager is down — confirm with the manager API directly (`authenticate`/`info`/`stats` returning 200) before chasing connectivity.
- In air-gapped Wazuh deployments, expect the app modules to be gated by the internet-dependent update/version check; plan analysis workflows around Discover from the start.
- The limitation affects only the dashboard convenience layer — data collection, agent enrollment, rule processing, and alerting are unaffected.

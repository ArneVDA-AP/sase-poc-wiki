---
title: "Finding: Wazuh Dashboard Apps section fails in air-gapped environment"
tags: [finding, wazuh, dashboard, air-gap]
---

# Finding: Wazuh Dashboard Apps section fails in air-gapped environment

**Component:** [Wazuh](../components/wazuh.md)  
**Severity:** Gotcha (known limitation)

## What happened

The Wazuh Dashboard Apps section is unreachable. Clicking the "API" section triggers a connection check to external Wazuh update servers, which fails in the air-gapped sandbox environment. The UI shows a perpetual loading spinner with no error message or timeout indication.

## Root cause

The dashboard performs a version/UUID health check against Wazuh's external API (`https://api.wazuh.com/...`) as part of the Apps section initialization. This check requires internet access. In the air-gapped lab environment, the request hangs indefinitely, blocking the entire Apps section from loading.

## Resolution / workaround

Use the **Discover** interface (direct OpenSearch query) as the primary analysis tool. Discover provides full event search, filtering, and visualization capabilities — sufficient for all PoC needs:

- Navigate to Discover in the Wazuh Dashboard sidebar
- Select the `wazuh-alerts-*` index pattern
- Use KQL or Lucene query syntax for filtering
- Create saved searches and visualizations as needed

The Apps section features (Agents overview, SCA, Vulnerability Detection dashboards) are not available, but the underlying data is fully accessible through Discover.

## Lessons

- Air-gapped deployments of Wazuh should expect the Apps section to be non-functional — plan analysis workflows around Discover from the start
- Discover is the primary analysis interface and is not affected by the external API dependency
- This limitation does not affect data collection, agent enrollment, or alerting — only the dashboard convenience layer

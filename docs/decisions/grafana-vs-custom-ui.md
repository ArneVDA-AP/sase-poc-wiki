---
title: "Decision: Grafana over a custom React UI"
tags: [decision, grafana, telemetry]
---

# Decision: Grafana over a custom React UI

**Status:** Implemented (PoC-validated on the parallel stack — sandbox integration pending)  
**Date:** April 2026 (Telemetry build v1)

## Context

The original telemetry plan was to build a bespoke React/Next.js frontend for the monitoring stack — a custom dashboard to view logs and metrics from the GNS3 lab and to start/stop GNS3 nodes. Before committing weeks of frontend work, the team re-evaluated whether a custom UI added anything over an off-the-shelf tool.

## Options considered

| Option | Pro | Con |
|--------|-----|-----|
| **Custom React/Next.js UI** | Full control over layout; could embed GNS3 node start/stop controls in one screen | Weeks of work; ends up a worse version of features Grafana already ships (log tailing, metric graphs, alerting); more code, more containers, more maintenance; the only unique feature (node start/stop) is already covered by the GNS3 web interface |
| **Grafana** | Industry standard, free; log tailing (Loki), metric graphs (Prometheus), alerting built in; dashboards-as-JSON survive container rebuilds and are version-controllable | No bespoke node start/stop controls in-dashboard (delegated to GNS3's own web UI) |

## Decision

Grafana, with no custom frontend.

The decisive reasoning: Grafana already provides everything the telemetry stack needs — log tailing, metric graphs, and alerting. The only feature a custom UI would have added is starting and stopping GNS3 nodes from the dashboard, and GNS3 already exposes its own web interface for that. Building a custom React app would have produced a weaker version of Grafana's existing capabilities while adding code, containers, and maintenance burden.

Choosing Grafana meant less code, fewer containers, less maintenance, and a faster result.

## Consequences

- Dashboards are authored as JSON files and provisioned at Grafana startup — they survive container rebuilds and are version-controllable. The trade-off is that provisioned dashboards and datasources are **read-only in the UI**; edits must go through the YAML/JSON files plus a container restart.
- GNS3 node start/stop is **not** integrated into the telemetry UI. Operators use the GNS3 web interface for node control and Grafana purely for observability.
- The decision narrowed the stack to proven, maintained tools (Grafana + Prometheus + Loki), which made the later migration from poc-1a to mgmt01 simpler — only forwarders and datasources had to move, not a hand-written application.

See also: [Component: Telemetry Stack](../components/telemetry-stack.md)

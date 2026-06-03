---
title: "Decision: Three-Layer CASB Architecture"
tags: [decision, casb, sase, nats, wazuh, squid]
---

# Decision: Three-Layer CASB Architecture

**Status:** Implemented  
**Date:** May-June 2026 (Verslag32-Verslag39)

## Context

The SASE architecture requires Cloud Access Security Broker (CASB) functionality as defined by Gartner's CASB sub-function framework. A single enforcement model cannot cover all required time horizons: request-time blocking, post-event audit response, and real-time cross-component correlation. The architecture must combine multiple enforcement layers to achieve meaningful CASB coverage within PoC scope.

## Decision

A three-layer CASB model is implemented, each layer covering a different enforcement time horizon:

**Layer 1 — Inline (Squid SWG pipeline):**
Request-time enforcement. Squid processes every HTTP/HTTPS request through the ICAP pipeline, applying URL filtering and identity-based blocking. When a user's request hits a blocked category or matches a DLP pattern, it is denied before the response is returned. This layer operates at the millisecond scale — the user sees an immediate block page.

**Layer 2 — API-mode (Wazuh + Microsoft Graph API):**
Post-event audit enforcement. Wazuh ingests Microsoft 365 audit events via the Graph API and applies detection rules. When a policy violation is detected (e.g., mass file download from SharePoint, external sharing of sensitive documents), Wazuh Active Response triggers automated remediation — typically within minutes of the event occurring. This layer covers SaaS activity that does not pass through the proxy.

**Layer 3 — Real-time (NATS + Control Daemon):**
Cross-component threat scoring and quarantine. The Control Daemon subscribes to events from multiple sources (Wazuh alerts, Squid deny logs, Suricata IDS alerts) via NATS JetStream and maintains a per-user threat score. When the score exceeds a threshold, the daemon issues a quarantine command to NetBird via API, isolating the peer from the network. This layer operates at the seconds scale and correlates signals that no single component can see alone.

## Consequences

- Each layer covers a different time horizon: request-time (milliseconds), audit-response (minutes), cross-component correlation (seconds).
- The three layers together provide approximately 15-25% Gartner CASB sub-function coverage. This is sufficient for PoC scope and demonstrates the architectural pattern without claiming full CASB compliance.
- Layer 1 requires the full SWG pipeline (Squid + c-icap + ClamAV + Python DLP) to be operational.
- Layer 2 requires Wazuh integration with Microsoft Graph API and appropriate M365 audit logging configuration.
- Layer 3 requires NATS JetStream, the Control Daemon, and NetBird API access to be operational simultaneously.
- Adding new CASB sub-functions means extending one of the three layers rather than building a fourth — the architecture is designed to be extensible within the existing model.

See also: [Component: Squid](../components/squid.md), [Component: Wazuh](../components/wazuh.md), [Component: Control Daemon](../components/control-daemon.md), [Component: NATS](../components/nats-jetstream.md), [Concept: SASE](../concepts/sase.md)

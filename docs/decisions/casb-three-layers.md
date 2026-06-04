---
title: "Decision: Three-Layer CASB Architecture"
tags: [decision, casb, sase, nats, wazuh, squid]
---

# Decision: Three-Layer CASB Architecture

**Status:** Implemented  
**Date:** May-June 2026 (Verslag32-Verslag39)

## Context

The SASE architecture requires Cloud Access Security Broker (CASB) functionality: SWG-anchored control over cloud-application use, integrated with identity. A single enforcement model cannot cover all required time horizons: request-time blocking, post-event audit response, and real-time cross-component correlation. The architecture must combine multiple enforcement layers to achieve meaningful CASB coverage within PoC scope.

## Decision

A three-layer CASB model is implemented, each layer covering a different enforcement time horizon:

**Layer 1 — Inline (Squid SWG pipeline):**
Request-time enforcement. Squid processes every HTTP/HTTPS request through the ICAP pipeline, applying URL filtering and identity-based blocking. When a user's request hits a blocked category or matches a DLP pattern, it is denied before the response is returned. This layer operates at the millisecond scale — the user sees an immediate block page.

**Layer 2 — API-mode (Office 365 Management Activity API + Wazuh + Microsoft Graph API):**
Post-event audit enforcement. A custom producer (`o365_producer`) polls the Office 365 Management Activity API for SharePoint audit events and publishes them to NATS; Wazuh rules (the 100600-family) then detect policy violations — anonymous link creation, anyone-scope sharing links, and guest sharing. On a violation, Wazuh Active Response scripts (`sharepoint_remediate.sh`, `guest_remediate.sh`) revoke the offending link via the Microsoft Graph API, behind an ENFORCE gate that defaults to detect-only. This layer covers SaaS activity that does not pass through the proxy; it operates within minutes (the Management Activity API has up to ~1 h latency and no SLA). The revoke chain is proven to HTTP-204 via an offline stub; live revocation against a real OneDrive file is pending test-account provisioning.

**Layer 3 — Real-time (NATS + Control Daemon):**
Cross-component threat scoring and quarantine. The Control Daemon subscribes to detection events on the NATS bus (`security.alert.>`) and dispatches on the `producer` field. Only malware (c-icap RESPMOD) and DLP events accrue threat score — proxy and IDS events are log-only (ambient proxy noise would accrue false positives; Suricata IDS carries no overlay attribution, so C2 response is deferred to Zeek/RITA). It maintains a per-peer threat score with sliding-window decay; when a peer crosses the quarantine threshold, the daemon strips it from its policy-bearing persona groups via the NetBird Groups API, and deny-by-default isolates it. The strip is scoped to persona groups, so infrastructure peers (`Core-Services`) stay reachable. This layer operates at the seconds scale and is reversible. The full dry-run → ENFORCE → reversible chain was validated end-to-end on test peer docent1 (V35): an EICAR malware event scored 80/80, the peer was quarantined within seconds and lost connectivity, then was restored on unquarantine.

## Consequences

- Each layer covers a different time horizon: request-time (milliseconds), audit-response (minutes), cross-component correlation (seconds).
- Rather than scoring against the full Gartner CASB framework (which would understate the work), the model is assessed against the project's CASB rubric — SWG-anchored cloud-app control. It covers the rubric's core well: **identity-integrated cloud-app blocking** (Layer 1, per-persona via the Identity Bridge and Entra ID), **group-scoped policies** (persona groups drive the inline and real-time layers), and **test results with a validated enforcement chain** (Layer 3, docent1/EICAR, V35). The thinner dimensions — context/risk/device-based blocking and explicit bypass testing — are the focus of the improvements below.
- Layer 1 requires the full SWG pipeline (Squid + c-icap + ClamAV + Python DLP) to be operational.
- Layer 2 requires the `o365_producer` polling the Office 365 Management Activity API, Wazuh rules plus Active Response, and Microsoft Graph API access for remediation; live revocation additionally needs test-account OneDrive provisioning.
- Layer 3 requires NATS JetStream, the Control Daemon, and NetBird API access to be operational simultaneously.
- Adding new CASB sub-functions means extending one of the three layers rather than building a fourth — the architecture is designed to be extensible within the existing model.

## Possible improvements

Not yet implemented — candidate enrichments that would strengthen the thinner rubric dimensions (context-based blocking and bypass testing), kept within the existing open-source stack:

- **Tenant Restrictions v2 via SSL-Bump header injection.** The organisation sanctions a single cloud suite (M365), so Squid (already doing SSL Bump) could inject `Restrict-Access-To-Tenants` / `Restrict-Access-Context` headers on Microsoft-login traffic — corporate sign-in allowed, personal/other-tenant sign-in blocked. A real context-based control at ~8 lines of `request_header_add`, directly serving the rubric's "dynamic/context-based cloud-app blocking" criterion.
- **Shadow-IT discovery from Squid access logs.** Squid logs already ship to Wazuh; classifying destination domains/SNI against a curated cloud-app list yields a per-user "discovered SaaS apps + risk" view on the Wazuh dashboard, with no new component. A curated app→domain list is more defensible for grading than a vendor catalogue — every entry is explainable and bypass-resistance (SNI + DNS-RPZ + IP) can be demonstrated.

See also: [Component: Squid](../components/squid.md), [Component: Wazuh](../components/wazuh.md), [Component: Control Daemon](../components/control-daemon.md), [Component: NATS](../components/nats-jetstream.md), [Concept: SASE](../concepts/sase.md)

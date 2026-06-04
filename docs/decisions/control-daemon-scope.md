---
title: "Decision: IDS Correlation Branch Removed from Control Daemon"
tags: [decision, control-daemon, ids, nats]
---

# Decision: IDS Correlation Branch Removed from Control Daemon

**Status:** Implemented  
**Date:** June 2026 (Verslag35)

## Context

The Control Daemon's threat scoring model initially included an IDS correlation branch designed to respond to C2 (Command and Control) beacon detection. When Suricata fired an IDS alert matching a C2 signature, the daemon would correlate the alert with user identity and trigger quarantine. This branch was added alongside the malware branch (which responds to ClamAV/ICAP malware detections) and the DLP branch (which responds to data exfiltration patterns).

During implementation, analysis revealed that the IDS correlation branch duplicated the quarantine capability already provided by the malware branch, but with worse attribution. The malware branch natively maps overlay IP addresses to peer identity through NetBird's API, providing clean user attribution. The IDS correlation branch would need to perform the same mapping but added no unique enforcement action — both branches result in the same quarantine outcome.

## Options considered

| Option | Pro | Con |
|--------|-----|-----|
| **Keep IDS correlation branch** | Adds explicit C2 beacon response capability; separate scoring weight for IDS events | Duplicates malware branch quarantine capability with worse attribution. C2 beacon detection is more naturally Zeek/RITA scope (behavioral analysis over time), not bespoke single-event correlation |
| **Remove IDS correlation branch** | Simpler scoring model; malware branch already covers real-time quarantine with native attribution; C2 response deferred to proper tooling (Zeek/RITA) | IDS events become log-only in the daemon — no automated response to IDS alerts |

## Decision

The IDS correlation branch is removed from the Control Daemon. IDS events from Suricata are still ingested via NATS JetStream but contribute only to logging, not to the threat score or automated quarantine actions.

The malware branch covers the same real-time quarantine capability with native attribution: when ClamAV detects malware in the ICAP pipeline, the daemon resolves the overlay IP to a NetBird peer identity and issues quarantine. C2 beacon response is deferred to Zeek/RITA, which performs behavioral analysis over time rather than single-event correlation.

Additionally, `proxy_block` events were removed from the scoring model. Ambient operating system noise (Windows telemetry, update checks, certificate revocation lookups) caused frequent proxy blocks that inflated threat scores without indicating genuine threats, leading to false positive quarantine triggers.

## Consequences

- IDS alerts from Suricata are log-only in the Control Daemon. They appear in NATS and in daemon logs but do not affect the threat score or trigger quarantine.
- The threat scoring model is simplified to two active branches: malware (ClamAV detections via ICAP) and DLP (data exfiltration patterns).
- `proxy_block` events are excluded from scoring. Only explicit malware detections and DLP violations contribute to the threat score.
- C2 beacon detection and response remains an open capability gap. If Zeek/RITA is added to the architecture in the future, C2 response could be reintroduced through that pathway rather than through the Control Daemon.
- The false positive rate from ambient OS noise is eliminated, making quarantine actions more reliable indicators of genuine threats.

See also: [Component: Control Daemon](../components/control-daemon.md), [Component: Suricata](../components/suricata.md), [Component: NATS](../components/nats-jetstream.md), [Decision: Three-Layer CASB Architecture](casb-three-layers.md)

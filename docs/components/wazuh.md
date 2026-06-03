---
title: "Wazuh"
tags: [wazuh, siem, nats, docker, wazuh-siem]
---

# Wazuh

**Role:** SIEM on mgmt01 Docker — receives events via the NATS→Wazuh forwarder (bus producers) and via direct Wazuh agent on pop01 (host feeds).  
**Version:** Wazuh v4.14.5 (`wazuh-docker`, tag-pinned)  
**Config location:** mgmt01 Docker, Wazuh stack (manager + indexer + dashboard)

## How it works in this stack

Wazuh provides forensic logging and API-mode CASB enforcement. It operates on a dual-write architecture: detection components write to logfiles (Wazuh agent picks up) AND publish to NATS (control daemon consumes) independently. Neither path depends on the other.

The NATS→Wazuh forwarder is a separate Docker container that consumes `security.alert.>` events from the NATS bus and writes them to the Wazuh manager socket as JSON log entries. Custom Wazuh rules then parse, classify, and index these events.

For CASB Layer 2, the M365 Activity API producer publishes SharePoint/OneDrive audit events to NATS. Wazuh rule 100600-family detects policy violations (anonymous shares, external shares). Active Response scripts revoke sharing links via Microsoft Graph API.

## Configuration

- **Dashboard:** `192.168.122.23:5601` (port 443 occupied by NetBird Caddy). Discover interface works fully. App API section unreachable due to air-gap connection check. See [Finding: Wazuh dashboard air-gate](../findings/wazuh-dashboard-airgate.md).
- **pop01 agent:** Enrolled as ID 001 Active. Delivers host feeds (audit, configd, filter, kernel, pkg). Suricata/Squid/c-icap/Unbound excluded from agent feeds — they have NATS producers.
- **Agent settings:** `intrusion_detection_events=false` + `active_response=false` (double-ingest guard + enforcement guard)
- **Rule-ID map:**
  - 100500–100540: Bus producers (proxy/IDS/DLP/malware/DNS-RPZ)
  - 100600-family: M365/CASB
- **Critical:** Always use `wazuh-control restart`, never `docker restart` (bind-mounts are overwritten in-place).

## Integration points

| Interface | Direction | Details |
|-----------|-----------|---------|
| NATS→Wazuh forwarder | Inbound | Consumes `security.alert.>`, writes to manager socket |
| pop01 Wazuh agent | Inbound | Agent ID 001, host feeds |
| M365 Activity API | Inbound (via o365_producer) | SharePoint/OneDrive audit events → NATS → forwarder |
| Microsoft Graph API | Outbound (Active Response) | Revoke sharing links (HTTP DELETE → 204) |

## Known issues / gotchas

- **glibc x86-64-v2 requirement:** Wazuh indexer requires Haswell+ CPU features. Standard QEMU `kvm64` CPU model crashes. Fix: set QEMU CPU model to `host` in GNS3 node settings. See [Finding: Wazuh CPU glibc](../findings/wazuh-cpu-glibc.md).
- **Dashboard air-gate:** App section unreachable — API connection check tries external Wazuh update servers, fails in air-gapped sandbox. Discover works fine. See [Finding: Wazuh dashboard air-gate](../findings/wazuh-dashboard-airgate.md).
- **jq dependency:** Active Response scripts require jq. Installed via `install_deps.sh` entrypoint (V39).

## Related

- [Component: NATS JetStream](nats-jetstream.md)
- [Component: Control Daemon](control-daemon.md)
- [Decision: CASB Three Layers](../decisions/casb-three-layers.md)

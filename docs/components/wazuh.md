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

The NATS→Wazuh forwarder is a separate Docker container — a durable PULL consumer (DeliverPolicy.NEW, AckPolicy.EXPLICIT) — that consumes `security.alert.>` events from the NATS bus and writes them as NDJSON to the shared `wazuh_nats_ingest` volume (`/ingest/security_alerts.json`). The Wazuh manager tails that file via a `<localfile><log_format>json</log_format></localfile>` block; custom Wazuh rules then parse, classify, and index the events. (Writing to the manager socket and posting to the Wazuh API were both evaluated and rejected — the localfile JSON tail is the working ingest path.)

For CASB Layer 2, the `o365_producer` polls the Office 365 Management Activity API and publishes SharePoint audit events to NATS (`security.alert.casb`). The Wazuh rule 100600-family detects policy violations (anonymous link creation, anyone-scope sharing links, guest sharing). Active Response scripts (`sharepoint_remediate.sh`, `guest_remediate.sh`) revoke sharing links via the Microsoft Graph API, behind an ENFORCE gate that defaults to detect-only. The revoke chain is proven to HTTP-204 via an offline stub; live revocation against a real OneDrive file is pending test-account provisioning (an external Microsoft dependency).

## Configuration

- **Dashboard:** `192.168.122.23:5601` (port 443 occupied by NetBird Caddy). Fully operational (fixed 2 juni 2026). `Error checking updates` CTI-500 remains in air-gap but is cosmetic and non-blocking in 4.14.5+. See [Finding: Wazuh dashboard air-gate](../findings/wazuh-dashboard-airgate.md).
- **pop01 agent:** Enrolled as ID 001 Active. Delivers host feeds (audit, configd, filter, kernel, pkg). Suricata/Squid/c-icap/Unbound excluded from agent feeds — they have NATS producers.
- **Agent settings:** `intrusion_detection_events=false` + `active_response=false` (double-ingest guard + enforcement guard)
- **Rule-ID map:**
  - 100500–100540: Bus producers (proxy/IDS/DLP/malware/DNS-RPZ)
  - 100600-family: M365/CASB
- **Critical:** Always use `wazuh-control restart`, never `docker restart` (bind-mounts are overwritten in-place).

## Integration points

| Interface | Direction | Details |
|-----------|-----------|---------|
| NATS→Wazuh forwarder | Inbound | Durable PULL consumer of `security.alert.>`; writes NDJSON to shared `wazuh_nats_ingest` volume (manager tails it via `<localfile>`) |
| pop01 Wazuh agent | Inbound | Agent ID 001, host feeds |
| M365 Activity API | Inbound (via o365_producer) | SharePoint audit events → `security.alert.casb` → NATS → forwarder |
| Microsoft Graph API | Outbound (Active Response) | Revoke sharing links (HTTP DELETE → 204); behind ENFORCE gate (detect-only default); live revoke pending OneDrive provisioning |

## Known issues / gotchas

- **glibc x86-64-v2 requirement:** Wazuh indexer requires Haswell+ CPU features. Standard QEMU `kvm64` CPU model crashes. Fix: set QEMU CPU model to `host` in GNS3 node settings. See [Finding: Wazuh CPU glibc](../findings/wazuh-cpu-glibc.md).
- **Dashboard air-gate (resolved 2 juni 2026):** Root cause was an empty manager UUID in `global.db` (wiped by `docker compose down -v`), not the air-gap CTI check. Fixed via in-place bump to 4.14.5 (no `-v`), preserving the UUID. The `Error checking updates` CTI-500 remains (air-gap) but is non-blocking in 4.14.5+. Never use `down -v` for Wazuh version work. See [Finding: Wazuh dashboard air-gate](../findings/wazuh-dashboard-airgate.md).
- **jq dependency:** Active Response scripts require jq. Installed via `install_deps.sh` entrypoint (V39).

## Related

- [Component: NATS JetStream](nats-jetstream.md)
- [Component: Control Daemon](control-daemon.md)
- [Decision: CASB Three Layers](../decisions/casb-three-layers.md)

---
title: "NATS JetStream"
tags: [nats, jetstream, event-bus, docker, nats-jetstream]
---

# NATS JetStream

**Role:** Central event bus on mgmt01 connecting isolated detection silos (Suricata, Squid, DLP, ClamAV, DNS-RPZ, Identity Bridge) into a coherent, reactive system.  
**Version:** NATS 2.14.1 (`nats:2.14-alpine` — empirically verified; Addendum J listed 2.12.6, outdated)  
**Config location:** mgmt01 Docker, secrets via env interpolation (no plaintext in committed config)

## How it works in this stack

NATS JetStream provides durable, at-least-once delivery for security events across the stack. Each detection component publishes structured JSON events to subject-specific topics. Two independent consumers process these events:

1. **Control Daemon** — real-time threat scoring and quarantine decisions
2. **NATS→Wazuh Forwarder** — bridges events to Wazuh for SIEM indexing and forensic analysis

The dual-write architecture ensures neither path depends on the other. A Wazuh outage does not affect real-time quarantine; a control daemon outage does not affect SIEM logging.

## Configuration

- **Auth model:** `accounts{}` — required for JetStream `$JS.API.>` access. The `authorization{}` model does not grant JetStream API access (empirically confirmed, V32).
- **Store dir:** `/data` — NATS appends `jetstream/` automatically. Never use `/data/jetstream` (causes double nesting). See [Finding: NATS store dir](../findings/nats-store-dir.md).
- **Docker network:** `sase-internal` (external, shared with all CASB containers)
- **Port:** 4222 (client, bound to 192.168.122.23), 8222 (HTTP monitoring, localhost only)

## Subject hierarchy

| Subject | Producer | Consumer |
|---------|----------|----------|
| `security.alert.ids` | Suricata (pop01) | Control Daemon, Wazuh forwarder |
| `security.alert.proxy` | Squid (pop01) | Control Daemon, Wazuh forwarder |
| `security.alert.dlp` | Python DLP (mgmt01) | Control Daemon, Wazuh forwarder |
| `security.alert.malware` | c-icap/ClamAV (pop01) | Control Daemon, Wazuh forwarder |
| `security.alert.dns` | DNS-RPZ producer (pop01) | Wazuh forwarder |
| `identity.login` | Identity Bridge (mgmt01) | Control Daemon |
| `identity.group_change` | Identity Bridge (mgmt01) | Control Daemon |
| `control.quarantine` | Control Daemon (mgmt01) | — (internal) |

## Integration points

| Interface | Direction | Details |
|-----------|-----------|---------|
| Detection producers (pop01) | Inbound | TCP 4222 via management LAN |
| Detection producers (mgmt01) | Inbound | Docker `sase-internal` network |
| Control Daemon | Outbound (consumer) | Durable consumer on `security.alert.>` + `identity.>` |
| Wazuh Forwarder | Outbound (consumer) | Durable consumer on `security.alert.>`, writes to Wazuh manager socket |

## Known issues / gotchas

- **Store dir double nesting:** See [Finding: NATS store dir](../findings/nats-store-dir.md).
- **Container recreate required:** `docker compose restart` after config changes does not pick up new environment variables. Use `docker compose up -d --force-recreate`.
- **Producers do not need `$JS.API.>` permissions:** Only subject publish + `_INBOX.>` are required for producers.

## Related

- [Decision: NATS accounts auth](../decisions/nats-accounts-auth.md)
- [Component: Control Daemon](control-daemon.md)
- [Component: Wazuh](wazuh.md)
- [Component: Identity Bridge](identity-bridge.md)

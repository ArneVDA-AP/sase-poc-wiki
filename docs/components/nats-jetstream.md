---
title: "NATS JetStream"
tags: [nats, jetstream, event-bus, docker, nats-jetstream]
---

# NATS JetStream

**Role:** Central event bus on mgmt01 connecting isolated detection silos (Suricata, Squid, DLP, ClamAV, DNS-RPZ, Identity Bridge) into a coherent, reactive system.  
**Version:** NATS 2.14.1 (`nats:2.14.1-alpine` — empirically verified; Addendum J listed 2.12.6, outdated)  
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
| `security.alert.casb` | o365 producer (mgmt01) | Wazuh forwarder |
| `identity.peer.connected` | Identity Bridge (mgmt01) | Control Daemon |
| `identity.peer.disconnected` | Identity Bridge (mgmt01) | Control Daemon |
| `identity.multi_persona` | Identity Bridge (mgmt01) | Control Daemon (zero-trust anomaly; SIEM rule planned) |

## JetStream streams

Five streams were created (V32, confirmed operational):

| Stream | Subjects | Storage | Max Age | Max Size |
|--------|----------|---------|---------|----------|
| SECURITY_ALERTS | `security.alert.*` | file | 168h (7d) | 1 GiB |
| THREAT_IOC | `threat.ioc.new` | file | 2160h (90d) | 512 MiB |
| POLICY_UPDATE | `policy.update` | file | 720h (30d) | 256 MiB |
| SESSION_CONTEXT | `session.context.>` | memory | 24h | 128 MiB |
| IDENTITY_EVENTS | `identity.>` | file | 168h (7d) | 256 MiB |

## Integration points

| Interface | Direction | Details |
|-----------|-----------|---------|
| Detection producers (pop01) | Inbound | TCP 4222 via management LAN |
| Detection producers (mgmt01) | Inbound | Docker `sase-internal` network |
| Control Daemon | Outbound (consumer) | Durable + DeliverPolicy.NEW consumer on `security.alert.>` (survives restart, no replay of historical events); ephemeral + DeliverPolicy.ALL consumer on `identity.>` (rebuilds the in-memory identity map each start) |
| Wazuh Forwarder | Outbound (consumer) | Durable PULL consumer (DeliverPolicy.NEW, AckPolicy.EXPLICIT) on `security.alert.>`; writes NDJSON to the shared `wazuh_nats_ingest` volume tailed by the Wazuh manager (localfile) |

## Known issues / gotchas

- **Store dir double nesting:** See [Finding: NATS store dir](../findings/nats-store-dir.md).
- **Container recreate required:** `docker compose restart` after config changes does not pick up new environment variables. Use `docker compose up -d --force-recreate`.
- **Producers do not need `$JS.API.>` permissions:** Only subject publish + `_INBOX.>` are required for producers.
- **Producer persistence (pop01):** The four pop01 producers (Suricata, Squid, DNS-RPZ, c-icap/ClamAV) are reboot-persistent via the `nats_producers` rc.d service (`sysrc nats_producers_enable=YES`), which starts them from the venv at `/usr/local/etc/nats-producers/venv`. Without it the producers do not survive a pop01 reboot and the event bus goes deaf. The Docker volume only persists the bus's own JetStream store, not the producers. See [Runbook 10: NATS JetStream](../runbooks/10-nats-jetstream.md).

## Related

- [Decision: NATS accounts auth](../decisions/nats-accounts-auth.md)
- [Component: Control Daemon](control-daemon.md)
- [Component: Wazuh](wazuh.md)
- [Component: Identity Bridge](identity-bridge.md)

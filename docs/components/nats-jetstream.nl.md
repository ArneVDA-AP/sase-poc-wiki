---
title: "NATS JetStream"
tags: [nats, jetstream, event-bus, docker, nats-jetstream]
---

# NATS JetStream

**Rol:** Centrale event bus op mgmt01 die geisoleerde detectiesilo's (Suricata, Squid, DLP, ClamAV, DNS-RPZ, Identity Bridge) verbindt tot een samenhangend, reactief systeem.  
**Versie:** NATS 2.14.1 (`nats:2.14-alpine` — empirisch geverifieerd; Addendum J vermeldde 2.12.6, verouderd)  
**Configuratielocatie:** mgmt01 Docker, secrets via env-interpolatie (geen plaintext in gecommitte configuratie)

## Hoe het werkt in deze stack

NATS JetStream biedt duurzame, at-least-once aflevering voor beveiligingsgebeurtenissen over de gehele stack. Elke detectiecomponent publiceert gestructureerde JSON-events naar onderwerpspecifieke topics. Twee onafhankelijke consumers verwerken deze events:

1. **Control Daemon** — real-time threat scoring en quarantainebeslissingen
2. **NATS→Wazuh Forwarder** — stuurt events door naar Wazuh voor SIEM-indexering en forensische analyse

De dual-write-architectuur garandeert dat geen van beide paden afhankelijk is van het andere. Een Wazuh-storing beinvloedt de real-time quarantaine niet; een uitval van de control daemon beinvloedt de SIEM-logging niet.

## Configuratie

- **Auth-model:** `accounts{}` — vereist voor JetStream `$JS.API.>` toegang. Het `authorization{}`-model verleent geen JetStream API-toegang (empirisch bevestigd, V32).
- **Store dir:** `/data` — NATS voegt automatisch `jetstream/` toe. Gebruik nooit `/data/jetstream` (veroorzaakt dubbele nesting). Zie [Bevinding: NATS store dir](../findings/nats-store-dir.md).
- **Docker-netwerk:** `sase-internal` (extern, gedeeld met alle CASB-containers)
- **Poort:** 4222 (client, gebonden aan 192.168.122.23), 8222 (HTTP-monitoring, enkel localhost)

## Onderwerpenhierarchie

| Onderwerp | Producer | Consumer |
|-----------|----------|----------|
| `security.alert.ids` | Suricata (pop01) | Control Daemon, Wazuh forwarder |
| `security.alert.proxy` | Squid (pop01) | Control Daemon, Wazuh forwarder |
| `security.alert.dlp` | Python DLP (mgmt01) | Control Daemon, Wazuh forwarder |
| `security.alert.malware` | c-icap/ClamAV (pop01) | Control Daemon, Wazuh forwarder |
| `security.alert.dns` | DNS-RPZ producer (pop01) | Wazuh forwarder |
| `identity.login` | Identity Bridge (mgmt01) | Control Daemon |
| `identity.group_change` | Identity Bridge (mgmt01) | Control Daemon |
| `control.quarantine` | Control Daemon (mgmt01) | — (intern) |

## Integratiepunten

| Interface | Richting | Details |
|-----------|----------|---------|
| Detectieproducers (pop01) | Inkomend | TCP 4222 via management-LAN |
| Detectieproducers (mgmt01) | Inkomend | Docker `sase-internal` netwerk |
| Control Daemon | Uitgaand (consumer) | Durable consumer op `security.alert.>` + `identity.>` |
| Wazuh Forwarder | Uitgaand (consumer) | Durable consumer op `security.alert.>`, schrijft naar Wazuh manager-socket |

## Bekende problemen / valkuilen

- **Store dir dubbele nesting:** Zie [Bevinding: NATS store dir](../findings/nats-store-dir.md).
- **Container-herschepping vereist:** `docker compose restart` na configuratiewijzigingen neemt nieuwe omgevingsvariabelen niet over. Gebruik `docker compose up -d --force-recreate`.
- **Producers hebben geen `$JS.API.>` rechten nodig:** Enkel subject publish + `_INBOX.>` zijn vereist voor producers.

## Gerelateerd

- [Beslissing: NATS accounts auth](../decisions/nats-accounts-auth.md)
- [Component: Control Daemon](control-daemon.md)
- [Component: Wazuh](wazuh.md)
- [Component: Identity Bridge](identity-bridge.md)

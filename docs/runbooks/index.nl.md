---
title: "Runbooks"
tags: [runbook]
---

# Runbooks

Stapsgewijze handleidingen voor het opbouwen van de SASE PoC-stack van nul af aan. Elke runbook herstelt de lineaire opbouwvolgorde uit de originele implementatiedocumenten, met waarschuwingen en valkuilen op exact het moment dat ze zich voordoen.

**Gebruik:** Volg de runbooks op volgorde. Elke runbook vermeldt zijn vereisten (welke voorgaande runbooks eerst afgerond moeten zijn). Elke stap eindigt met een verificatie zodat je weet wanneer je door kunt gaan. Voor dieper conceptueel begrip kun je de links naar de bijbehorende component-, concept- en beslissingsartikelen volgen.

---

## Bouwvolgorde

```
01 Labomgeving
 └─► 02 ZTNA Overlay
      ├─► 03 Proxy & WPAD ─► 04 Malware & DLP
      │                      05 IDS (parallel met 04)
      ├─► 06 DNS Threat Intel
      ├─► 08 GroupSync (na 02)
      │    └─► 09 Identity Bridge (na 08)
      ├─► 10 NATS JetStream (na 03, 04, 05, 06)
      │    └─► 11 Wazuh (na 10)
      ├─► 07 Access Policy (na alles hierboven)
      └─► 12 Intune Endpoint (na 07)
```

## Runbook-overzicht

| # | Runbook | Node(s) | Status | Beschrijving |
|---|---------|---------|--------|--------------|
| 1 | [Labomgeving](01-lab-environment.nl.md) | GNS3-host (poc-1a) | Operationeel | Proxmox VM, GNS3 Server, topologie, IP-adressering, snapshots |
| 2 | [ZTNA Overlay](02-ztna-overlay.nl.md) | mgmt01 + alle peers | Operationeel | NetBird + Zitadel + Entra ID, groepen, ACL's, exit node, DNS |
| 3 | [Proxy & WPAD](03-proxy-wpad.nl.md) | pop01 + mgmt01 | Operationeel | Squid expliciete proxy, WPAD/PAC via Caddy, SSL Bump, URL-filtering |
| 4 | [Malware & DLP](04-malware-dlp.nl.md) | pop01 + mgmt01 | Operationeel | ClamAV/c-icap RESPMOD, YARA-regels, Python DLP REQMOD |
| 5 | [IDS](05-ids.nl.md) | pop01 | Operationeel | Suricata op WAN+LAN, Hyperscan, 79.620+ regels, drop/alert policy |
| 6 | [DNS Threat Intel](06-dns-threat-intel.nl.md) | mgmt01 + pop01 | Operationeel | ioc2rpz-feeds, BIND TSIG-intermediair, Unbound RPZ, 71.767 records |
| 7 | [Access Policy](07-access-policy.nl.md) | Entra ID + NetBird | Operationeel | Conditional Access (5 policies), posture checks, validatiescenario's |
| 8 | [GroupSync](08-groupsync.nl.md) | Entra ID + Zitadel + NetBird | Operationeel | JWT group sync, Entra ID token config, Zitadel Actions |
| 9 | [Identity Bridge](09-identity-bridge.nl.md) | mgmt01 + pop01 | Operationeel | FastAPI overlay-IP → persona-groep, Squid external_acl |
| 10 | [NATS JetStream](10-nats-jetstream.nl.md) | mgmt01 + pop01 | Operationeel | Event bus, producers, Control Daemon, Redis |
| 11 | [Wazuh](11-wazuh.nl.md) | mgmt01 + pop01 | Operationeel | SIEM-stack, NATS-forwarder, M365 Active Response |
| 12 | [Intune Endpoint](12-intune-endpoint.nl.md) | Entra ID + Intune | Geïmplementeerd | Certificaat, afgedwongen PAC, firewall-block-set, Teams-QoS, split-tunnel-Remediation; SITE01-enrollment |
| 13 | [Cosmos](13-cosmos.nl.md) | dc01 (parallelle stack) | PoC | Cosmos-install en per-app MFA (parallelle stack) |
| 14 | [Zeek/RITA](14-zeek-rita.nl.md) | mgmt01 (parallelle stack) | PoC | Hub/GRE-setup, Zeek-cluster, RITA-beacon-analyse (parallelle stack) |
| 15 | [RITA naar RPZ](15-rita-rpz-integration.nl.md) | mgmt01 + pop01 (parallelle stack) | PoC | Geautomatiseerde RITA naar ioc2rpz naar BIND naar Unbound-feed (parallelle stack) |
| 16 | [Telemetry](16-telemetry.nl.md) | mgmt01 (parallelle stack) | PoC | Grafana / Prometheus / Loki-stack deployen (parallelle stack) |

## Afhankelijkheidsgraph

| Runbook | Afhankelijk van |
|---------|-----------------|
| 01 Labomgeving | n.v.t. |
| 02 ZTNA Overlay | 01 |
| 03 Proxy & WPAD | 02 |
| 04 Malware & DLP | 03 (SSL Bump vereist) |
| 05 IDS | 03 (Squid operationeel), 01 (8 GB RAM) |
| 06 DNS Threat Intel | 02 (NetBird DNS-relay) |
| 07 Access Policy | 02, 03, 04, 05, 06 (volledige stack) |
| 08 GroupSync | 02 |
| 09 Identity Bridge | 08 (GroupSync voltooid) |
| 10 NATS JetStream | 03, 04, 05, 06 (alle producers operationeel) |
| 11 Wazuh | 10 (NATS operationeel) |
| 12 Intune Endpoint | 07 (managed device Entra-joined + Intune-enrolled) |

## Snelreferentie: belangrijkste poorten

| Service | Adres | Poort |
|---------|-------|-------|
| Squid-proxy | `100.70.154.79` | 3128 |
| Unbound DNS | `100.70.154.79` | 53 |
| BIND (TSIG secundair) | `127.0.0.1` | 53530 |
| ioc2rpz | `192.168.122.23` | 53 |
| Python DLP ICAP | `192.168.122.23` | 1345 |
| ClamAV/c-icap | `127.0.0.1` | 1344 |
| NetBird Dashboard | `netbird.sandbox.local` | 443 |
| ioc2rpz GUI | `ioc2rpz.sandbox.local` | 443 |
| WPAD PAC-bestand | `wpad.sandbox.local` | 80 |

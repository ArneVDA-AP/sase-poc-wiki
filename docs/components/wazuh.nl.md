---
title: "Wazuh"
tags: [wazuh, siem, nats, docker, wazuh-siem]
---

# Wazuh

**Rol:** SIEM op mgmt01 Docker — ontvangt events via de NATS→Wazuh forwarder (busproducers) en via de directe Wazuh-agent op pop01 (host-feeds).  
**Versie:** Wazuh v4.14.5 (`wazuh-docker`, tag-pinned)  
**Configuratielocatie:** mgmt01 Docker, Wazuh-stack (manager + indexer + dashboard)

## Hoe het werkt in deze stack

Wazuh voorziet in forensische logging en API-mode CASB-handhaving. Het werkt op een dual-write-architectuur: detectiecomponenten schrijven naar logbestanden (Wazuh-agent leest deze uit) EN publiceren naar NATS (control daemon consumeert) onafhankelijk van elkaar. Geen van beide paden is afhankelijk van het andere.

De NATS→Wazuh forwarder is een aparte Docker-container die `security.alert.>` events van de NATS-bus consumeert en ze als JSON-logregels naar de Wazuh manager-socket schrijft. Aangepaste Wazuh-regels parseren, classificeren en indexeren deze events vervolgens.

Voor CASB Laag 2 publiceert de M365 Activity API-producer SharePoint/OneDrive-auditevents naar NATS. Wazuh-regelfamilie 100600 detecteert beleidsovertredingen (anonieme shares, externe shares). Active Response-scripts trekken deellinks in via de Microsoft Graph API.

## Configuratie

- **Dashboard:** `192.168.122.23:5601` (poort 443 bezet door NetBird Caddy). Discover-interface werkt volledig. App API-sectie onbereikbaar door air-gap-verbindingscontrole. Zie [Bevinding: Wazuh dashboard air-gate](../findings/wazuh-dashboard-airgate.md).
- **pop01-agent:** Ingeschreven als ID 001 Active. Levert host-feeds (audit, configd, filter, kernel, pkg). Suricata/Squid/c-icap/Unbound uitgesloten van agent-feeds — deze hebben NATS-producers.
- **Agent-instellingen:** `intrusion_detection_events=false` + `active_response=false` (dubbel-ingest-bescherming + handhavingsbescherming)
- **Rule-ID-overzicht:**
  - 100500–100540: Busproducers (proxy/IDS/DLP/malware/DNS-RPZ)
  - 100600-familie: M365/CASB
- **Kritiek:** Gebruik altijd `wazuh-control restart`, nooit `docker restart` (bind-mounts worden in-place overschreven).

## Integratiepunten

| Interface | Richting | Details |
|-----------|----------|---------|
| NATS→Wazuh forwarder | Inkomend | Consumeert `security.alert.>`, schrijft naar manager-socket |
| pop01 Wazuh-agent | Inkomend | Agent ID 001, host-feeds |
| M365 Activity API | Inkomend (via o365_producer) | SharePoint/OneDrive-auditevents → NATS → forwarder |
| Microsoft Graph API | Uitgaand (Active Response) | Deellinks intrekken (HTTP DELETE → 204) |

## Bekende problemen / valkuilen

- **glibc x86-64-v2-vereiste:** Wazuh indexer vereist Haswell+ CPU-features. Het standaard QEMU `kvm64` CPU-model crasht. Oplossing: stel het QEMU CPU-model in op `host` in de GNS3-node-instellingen. Zie [Bevinding: Wazuh CPU glibc](../findings/wazuh-cpu-glibc.md).
- **Dashboard air-gate:** App-sectie onbereikbaar — API-verbindingscontrole probeert externe Wazuh-updateservers te bereiken, faalt in air-gapped sandbox. Discover werkt prima. Zie [Bevinding: Wazuh dashboard air-gate](../findings/wazuh-dashboard-airgate.md).
- **jq-afhankelijkheid:** Active Response-scripts vereisen jq. Geinstalleerd via `install_deps.sh` entrypoint (V39).

## Gerelateerd

- [Component: NATS JetStream](nats-jetstream.md)
- [Component: Control Daemon](control-daemon.md)
- [Beslissing: CASB Three Layers](../decisions/casb-three-layers.md)

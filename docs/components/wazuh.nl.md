---
title: "Wazuh"
tags: [wazuh, siem, nats, docker, wazuh-siem]
---

# Wazuh

**Rol:** SIEM op mgmt01 Docker dat events ontvangt via de NATS→Wazuh forwarder (busproducers) en via de directe Wazuh-agent op pop01 (host-feeds).  
**Versie:** Wazuh v4.14.5 (`wazuh-docker`, tag-pinned)  
**Configuratielocatie:** mgmt01 Docker, Wazuh-stack (manager + indexer + dashboard)

## Hoe het werkt in deze stack

Wazuh voorziet in forensische logging en API-mode CASB-handhaving. Het werkt op een dual-write-architectuur: detectiecomponenten schrijven naar logbestanden (Wazuh-agent leest deze uit) EN publiceren naar NATS (control daemon consumeert) onafhankelijk van elkaar. Geen van beide paden is afhankelijk van het andere.

De NATS→Wazuh forwarder is een aparte Docker-container (een durable PULL-consumer, DeliverPolicy.NEW, AckPolicy.EXPLICIT) die `security.alert.>` events van de NATS-bus consumeert en ze als NDJSON naar het gedeelde `wazuh_nats_ingest`-volume schrijft (`/ingest/security_alerts.json`). De Wazuh manager tailt dat bestand via een `<localfile><log_format>json</log_format></localfile>`-blok; aangepaste Wazuh-regels parseren, classificeren en indexeren de events vervolgens. (Rechtstreeks naar de manager-socket schrijven en naar de Wazuh-API posten zijn beide geëvalueerd en verworpen; de localfile-JSON-tail is het werkende ingestiepad.)

Voor CASB Laag 2 pollt de `o365_producer` de Office 365 Management Activity API en publiceert SharePoint-auditevents naar NATS (`security.alert.casb`). De Wazuh-regelfamilie 100600 detecteert beleidsovertredingen (anonieme link-aanmaak, anyone-scope deellinks, gast-sharing). Active Response-scripts (`sharepoint_remediate.sh`, `guest_remediate.sh`) trekken deellinks in via de Microsoft Graph API, achter een ENFORCE-gate die standaard op detect-only staat. De revoke-keten is bewezen tot HTTP-204 via een offline stub; live intrekking tegen een echt OneDrive-bestand wacht op provisioning van een testaccount (een externe Microsoft-afhankelijkheid).

## Configuratie

- **Dashboard:** `192.168.122.23:5601` (poort 443 bezet door NetBird Caddy). Volledig operationeel (opgelost 2 juni 2026). `Error checking updates` CTI-500 blijft aanwezig in air-gap maar is cosmetisch en non-blocking in 4.14.5+. Zie [Bevinding: Wazuh dashboard air-gate](../findings/wazuh-dashboard-airgate.md).
- **pop01-agent:** Ge-enrolld als ID 001 Active. Levert host-feeds (audit, configd, filter, kernel, pkg). Suricata/Squid/c-icap/Unbound uitgesloten van agent-feeds; deze hebben NATS-producers.
- **Agent-instellingen:** `intrusion_detection_events=false` + `active_response=false` (dubbel-ingest-bescherming + handhavingsbescherming)
- **Rule-ID-overzicht:**
  - 100500–100540: Busproducers (proxy/IDS/DLP/malware/DNS-RPZ)
  - 100600-familie: M365/CASB
- **Kritiek:** Gebruik altijd `wazuh-control restart`, nooit `docker restart` (bind-mounts worden in-place overschreven).

## Integratiepunten

| Interface | Richting | Details |
|-----------|----------|---------|
| NATS→Wazuh forwarder | Inkomend | Durable PULL-consumer van `security.alert.>`; schrijft NDJSON naar gedeeld `wazuh_nats_ingest`-volume (manager tailt het via `<localfile>`) |
| pop01 Wazuh-agent | Inkomend | Agent ID 001, host-feeds |
| M365 Activity API | Inkomend (via o365_producer) | SharePoint-auditevents → `security.alert.casb` → NATS → forwarder |
| Microsoft Graph API | Uitgaand (Active Response) | Deellinks intrekken (HTTP DELETE → 204); achter ENFORCE-gate (detect-only standaard); live revoke wacht op OneDrive-provisioning |

## Bekende problemen / valkuilen

- **glibc x86-64-v2-vereiste:** Wazuh indexer vereist Haswell+ CPU-features. Het standaard QEMU `kvm64` CPU-model crasht. Oplossing: stel het QEMU CPU-model in op `host` in de GNS3-node-instellingen. Zie [Bevinding: Wazuh CPU glibc](../findings/wazuh-cpu-glibc.md).
- **Dashboard air-gate (opgelost 2 juni 2026):** De werkelijke oorzaak was een lege manager-UUID in `global.db` (gewist door `docker compose down -v`), niet de air-gap CTI-controle. Opgelost via in-place bump naar 4.14.5 (zonder `-v`), waardoor de UUID bewaard bleef. De `Error checking updates` CTI-500 blijft (air-gap) maar is non-blocking in 4.14.5+. Gebruik nooit `down -v` voor Wazuh versie-werk. Zie [Bevinding: Wazuh dashboard air-gate](../findings/wazuh-dashboard-airgate.md).
- **jq-afhankelijkheid:** Active Response-scripts vereisen jq. Geinstalleerd via `install_deps.sh` entrypoint (V39).

## Gerelateerd

- [Component: NATS JetStream](nats-jetstream.md)
- [Component: Control Daemon](control-daemon.md)
- [Beslissing: CASB Three Layers](../decisions/casb-three-layers.md)

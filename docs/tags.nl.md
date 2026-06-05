---
title: "Gecontroleerd tag-vocabulaire"
---

# Gecontroleerd tag-vocabulaire

Alle wiki-pagina's gebruiken uitsluitend tags uit deze lijst. Een nieuwe tag wordt alleen toegevoegd wanneer een concept in minstens twee brondocumenten voorkomt en geen bestaande tag het dekt.

---

## Paginatype-tags

| Tag | Van toepassing op |
|-----|-------------------|
| `architecture` | Pagina's over systeemstructuur of topologie |
| `decision` | Beslissingsrecordpagina's (`decisions/`) |
| `finding` | Bevindingen- / valkuilpagina's (`findings/`) |
| `workaround` | Bevindingen met een workaround voor een bug of beperking |

---

## Componenttags

| Tag | Component |
|-----|-----------|
| `bind` | BIND 9.20, TSIG-geauthenticeerde secundaire zoneserver op pop01 |
| `c-icap` | c-icap server, ICAP-daemon die ClamAV aanstuurt op pop01 |
| `caddy` | Caddy, WPAD-server, TLS-terminator, reverse proxy op mgmt01 |
| `clamav` | ClamAV, malwarescanner en DLP-laag 1 op pop01 |
| `control-daemon` | Control Daemon, threat scoring + real-time quarantaine via NetBird API op mgmt01 |
| `docker` | Docker / Docker Compose, container runtime op mgmt01 |
| `identity-bridge` | Identity Bridge, FastAPI-service die overlay-IP's mapt naar Entra ID-persona groepen op mgmt01 |
| `ioc2rpz` | ioc2rpz, RPZ-zone-aggregator op mgmt01 |
| `nats-jetstream` | NATS JetStream, centrale event bus die detectiesilo's verbindt op mgmt01 |
| `netbird` | NetBird, WireGuard-gebaseerde ZTNA-overlay (inclusief Zitadel, Entra ID) |
| `opnsense` | OPNsense 25.1, firewall-OS op pop01 |
| `python` | Python ICAP-server, upload DLP-laag 2 op mgmt01 |
| `redis` | Redis, threat score-opslag + sessiestatus voor control daemon op mgmt01 |
| `squid` | Squid 6.x, expliciete HTTP/HTTPS-proxy op pop01 |
| `suricata` | Suricata 7.x, IDS op pop01 WAN + LAN-interfaces |
| `unbound` | Unbound, DNS-resolver met RPZ-afdwinging op pop01 |
| `wazuh` | Wazuh v4.14.5, SIEM met NATS-forwarder + M365 Active Response op mgmt01 |
| `zitadel` | Zitadel, OIDC IdP-broker tussen NetBird en Entra ID op mgmt01 |

---

## Protocol- en technologietags

| Tag | Concept |
|-----|---------|
| `dns` | Domain Name System, resolutie, zones, threat intelligence |
| `firewall` | Pakketfilter-firewall (OPNsense pf) |
| `icap` | Internet Content Adaptation Protocol, proxy-inspectie-offload |
| `ids` | Intrusion Detection System-modus (PCAP, alleen alert) |
| `ips` | Intrusion Prevention System-modus (inline, drop-capable) |
| `multipart` | `multipart/form-data`-encoding, upload-body-parsing |
| `network` | Algemene netwerken: routing, interfaces, segmenten |
| `pac` | Proxy Auto-Configuration, het JavaScript PAC-bestandsformaat |
| `proxy` | HTTP/HTTPS-proxy, expliciet of transparant |
| `reqmod` | ICAP REQMOD, request-side contentadaptatie |
| `respmod` | ICAP RESPMOD, response-side contentadaptatie |
| `rpz` | DNS Response Policy Zones, DNS-level domeinblokkering |
| `ssl-bump` | Squid SSL Bump, HTTPS-interceptie via on-the-fly CA |
| `tls` | TLS/SSL, encryptie, certificaten, trust stores |
| `wpad` | Web Proxy Auto-Discovery, browser proxy-configuratieprotocol |

---

## Protocol- en integratietags

| Tag | Concept |
|-----|---------|
| `entra-id` | Microsoft Entra ID, cloud identity provider (aplab.be-tenant) |
| `event-bus` | Event-driven architectuur, NATS publish/subscribe-patroon |
| `groupsync` | JWT group sync, Entra ID → Zitadel → NetBird-groepspropagatie |
| `intune` | Microsoft Intune, apparaatconformiteit en -beheer |
| `mdm` | Mobile Device Management, Intune-beheer van het endpoint |
| `oidc` | OpenID Connect, authenticatieprotocol (Entra ID → Zitadel → NetBird) |
| `sd-wan` | SD-WAN / Zero Trust Branch, WAN-connectiviteit en verkeersoptimalisatie |
| `testing` | Testscenario's, acceptatietests, demovalidatie |

---

## Beveiligingsconcepttags

| Tag | Concept |
|-----|---------|
| `antivirus` | Signature-gebaseerde malwaredetectie (ClamAV-databases) |
| `casb` | Cloud Access Security Broker, drielaags handhaving (inline, API, real-time) |
| `dlp` | Data Loss Prevention, detectie en blokkering van gevoelige data-exfiltratie |
| `fwaas` | Firewall as a Service, OPNsense + Suricata IDS |
| `identity` | Identiteitsgebaseerde toegangscontrole, gebruiker/groep-resolutie door de stack |
| `sase` | Secure Access Service Edge, het overkoepelende architectuurkader |
| `siem` | Security Information and Event Management (Wazuh) |
| `swg` | Secure Web Gateway, Squid + SSL Bump + ICAP-pipeline |
| `yara` | YARA-regels, patroonherkenningsengine voor malware- en DLP-detectie |
| `zero-trust` | Zero Trust-beveiligingsmodel, never trust, always verify; drie-gate model |
| `ztna` | Zero Trust Network Access, identiteitsgebaseerde tunnel (NetBird + Entra ID) |

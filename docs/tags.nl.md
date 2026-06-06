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
| `cosmos` | Cosmos, identity-aware application gateway met per-app MFA (parallel-stack PoC) |
| `docker` | Docker / Docker Compose, container runtime op mgmt01 |
| `gns3` | GNS3, lab-virtualisatieplatform (geneste QEMU/KVM op Proxmox) |
| `grafana` | Grafana, telemetry-dashboards in de observability-laag (parallel-stack PoC) |
| `identity-bridge` | Identity Bridge, FastAPI-service die overlay-IP's mapt naar Entra ID-persona groepen op mgmt01 |
| `ioc2rpz` | ioc2rpz, RPZ-zone-aggregator op mgmt01 |
| `loki` | Loki, log-aggregatiebackend in de observability-laag (parallel-stack PoC) |
| `nats-jetstream` | NATS JetStream, centrale event bus die detectiesilo's verbindt op mgmt01 |
| `netbird` | NetBird, WireGuard-gebaseerde ZTNA-overlay (inclusief Zitadel, Entra ID) |
| `opnsense` | OPNsense 25.1, firewall-OS op pop01 |
| `prometheus` | Prometheus, metrics-collectie in de observability-laag (parallel-stack PoC) |
| `python` | Python ICAP-server, upload DLP-laag 2 op mgmt01 |
| `redis` | Redis, threat score-opslag + sessiestatus voor control daemon op mgmt01 |
| `rita` | RITA, beaconing / C2 behavioral analysis over Zeek-logs, voedt RPZ (parallel-stack PoC) |
| `smartshield` | SmartShield, adaptieve WAF / anti-bot-engine van Cosmos (parallel-stack PoC) |
| `squid` | Squid 6.x, expliciete HTTP/HTTPS-proxy op pop01 |
| `suricata` | Suricata 7.x, IDS op pop01 WAN + LAN-interfaces |
| `unbound` | Unbound, DNS-resolver met RPZ-afdwinging op pop01 |
| `wazuh` | Wazuh v4.14.5, SIEM met NATS-forwarder + M365 Active Response op mgmt01 |
| `zeek` | Zeek, network security monitor voor protocol-analyse en behavioral logging (parallel-stack PoC) |
| `zitadel` | Zitadel, OIDC IdP-broker tussen NetBird en Entra ID op mgmt01 |

---

## Protocol- en technologietags

| Tag | Concept |
|-----|---------|
| `dns` | Domain Name System, resolutie, zones, threat intelligence |
| `firewall` | Pakketfilter-firewall (OPNsense pf) |
| `gre` | Generic Routing Encapsulation, tunnel die gespiegeld verkeer naar Zeek draagt (parallel-stack PoC) |
| `icap` | Internet Content Adaptation Protocol, proxy-inspectie-offload |
| `ids` | Intrusion Detection System-modus (PCAP, alleen alert) |
| `ips` | Intrusion Prevention System-modus (inline, drop-capable) |
| `mfa` | Multifactorauthenticatie, per-app-challenge afgedwongen door Cosmos (parallel-stack PoC) |
| `multipart` | `multipart/form-data`-encoding, upload-body-parsing |
| `network` | Algemene netwerken: routing, interfaces, segmenten |
| `pac` | Proxy Auto-Configuration, het JavaScript PAC-bestandsformaat |
| `proxy` | HTTP/HTTPS-proxy, expliciet of transparant |
| `reqmod` | ICAP REQMOD, request-side contentadaptatie |
| `respmod` | ICAP RESPMOD, response-side contentadaptatie |
| `reverse-proxy` | Reverse proxy, front-end-gateway die clientverbindingen termineert (Cosmos, Caddy) |
| `rpz` | DNS Response Policy Zones, DNS-level domeinblokkering |
| `ssl-bump` | Squid SSL Bump, HTTPS-interceptie via on-the-fly CA |
| `telemetry` | Telemetry / observability, metrics, logs, dashboards (Grafana + Prometheus + Loki) |
| `tls` | TLS/SSL, encryptie, certificaten, trust stores |
| `tproxy` | Transparante proxy (TPROXY), kernel-level interceptie voor all-traffic capture |
| `wpad` | Web Proxy Auto-Discovery, browser proxy-configuratieprotocol |

---

## Protocol- en integratietags

| Tag | Concept |
|-----|---------|
| `entra-id` | Microsoft Entra ID, cloud identity provider (aplab.be-tenant) |
| `event-bus` | Event-driven architectuur, NATS publish/subscribe-patroon |
| `groupsync` | JWT group sync, Entra ID → Zitadel → NetBird-groepspropagatie |
| `intune` | Microsoft Intune, device compliance en -beheer |
| `mdm` | Mobile Device Management, Intune-beheer van het endpoint |
| `oidc` | OpenID Connect, authenticatieprotocol (Entra ID → Zitadel → NetBird) |
| `sd-wan` | SD-WAN / Zero Trust Branch, WAN-connectiviteit en verkeersoptimalisatie |
| `testing` | Testscenario's, acceptatietests, demovalidatie |

---

## Beveiligingsconcepttags

| Tag | Concept |
|-----|---------|
| `antivirus` | Signature-gebaseerde malwaredetectie (ClamAV-databases) |
| `application-gateway` | Reverse proxy met identity gate en MFA als application-admission-laag |
| `beaconing` | Periodiek callback-patroon van gecompromitteerde hosts die C2-infrastructuur contacteren |
| `behavioral-analysis` | Beaconing / C2-detectie op gedrag in plaats van signature (RITA over Zeek-logs) |
| `c2` | Command-and-control, aanvallersinfrastructuur waar gecompromitteerde hosts naar beaconen |
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

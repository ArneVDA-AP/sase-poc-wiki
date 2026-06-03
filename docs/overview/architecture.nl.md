---
title: "Systeemarchitectuur"
tags: [sase, architecture, network, opnsense, netbird, squid, suricata, zero-trust]
---

# Systeemarchitectuur

Deze pagina beschrijft de volledige architectuur van de SASE PoC: nodes, netwerksegmenten, de control/data plane-splitsing en hoe alle beveiligingscomponenten met elkaar samenhangen. Elke componentpagina verwijst hiernaartoe voor architecturale context.

---

## 1. Ontwerpfilosofie: Castle & Moat → Edge-handhaving

Het project verwerpt het traditionele perimetermodel waarbij vertrouwen voortkomt uit netwerkpositie. In het traditionele model beschermt een firewall één grens; iedereen binnenin is vertrouwd. Dit werkt niet voor Atlascollege: 4 000 gebruikers op beheerde Windows-apparaten die vanuit thuis, cafés en treinen werken — er is geen "binnenste" meer.

In plaats daarvan vinden inspectie en vertrouwensbeslissingen plaats **op de PoP** (Point of Presence) — de edge-node tussen clients en internet. Vertrouwen wordt bepaald door drie onafhankelijk geëvalueerde factoren:

1. **Identiteit** — wie ben je? (Entra ID + MFA)
2. **Apparaatposture** — wat is de toestand van je apparaat? (Intune-conformiteit + Entra ID CA)
3. **Inhoud** — wat verstuur je? (SWG-inspectiepipeline)

Dit is het [Zero Trust drie-gate model](../concepts/zero-trust.md).

---

## 2. Nodes en hun rollen

| Node | Platform | Rol | IP (WAN/mgmt) | IP (overlay) |
|------|----------|-----|----------------|--------------|
| **pop01** | OPNsense 25.1 (FreeBSD) | Data plane PoP: firewall, proxy, IDS, DNS-resolver | `192.168.122.13` | `100.70.154.79` |
| **mgmt01** | Ubuntu 24.04 (Docker) | Management/control plane: NetBird-stack, ioc2rpz, DLP ICAP, WPAD | `192.168.122.23` | `100.70.135.241` |
| **dc01** | Ubuntu 24.04 | Datacentersimulatie — interne resource-doelwit | `10.0.0.100` (DC-LAN) | — |
| **site01** | VyOS | SASE-gateway voor remote site — WAN-connectiviteit + NAT | `192.168.122.33` | — |
| **sitepc01** | Windows 11 (Tiny11) | Remote site-endpoint — dual-NIC op Site-LAN, sandbox-enrolled | `172.16.10.x` | — |
| **mobile01** | Windows 11 (VMware, buiten GNS3) | Beheerde client — simuleert Intune-enrolled apparaat vanaf elke locatie | extern | `100.70.95.98` |

Alle nodes behalve mobile01 draaien in GNS3 op een Proxmox-gehoste Ubuntu-VM. Zie [GNS3 / Topologie](../components/gns3.md) voor de volledige infrastructuurlaag.

---

## 3. Netwerksegmenten

| Segment | CIDR | Doel | Verbonden nodes |
|---------|------|------|-----------------|
| **WAN / beheer** | `192.168.122.0/24` | GNS3 libvirt NAT; underlay voor al het inter-node-verkeer; GNS3 host internet | pop01, mgmt01, site01 (allemaal via vtnet0/ens3) |
| **DC-LAN** | `10.0.0.0/24` | Intern datacentersegment — simuleert on-premise resources | pop01 (vtnet1 = gateway `.1`), dc01 (`.100`) |
| **Site-LAN** | `172.16.10.0/24` | Remote site-segment | site01 (eth1 = gateway `.1`), sitepc01 (`.50`, geen OS) |
| **NetBird Overlay** | `100.64.0.0/10` (WireGuard mesh) | Versleutelde overlay — client-transport voor al het SASE-verkeer | pop01, mgmt01, mobile01 |

mobile01 bevindt zich **niet** op het WAN-segment. Het verbindt uitsluitend via de NetBird WireGuard-overlay, als simulatie van een echte externe gebruiker zonder directe verbinding naar het interne netwerk.

---

## 4. Control Plane vs Data Plane

Een bewuste architecturale splitsing die commerciële SASE weerspiegelt:

**Management plane (mgmt01):**
- NetBird management-stack (Zitadel OIDC, management server, Caddy reverse proxy) — via Docker Compose
- ioc2rpz — threat intelligence-aggregatie en RPZ-zonepublicatie
- Caddy — WPAD-server, ioc2rpz GUI-proxy
- Identity Bridge (FastAPI) — mapt overlay-IP's naar Entra ID persona-groepen voor Squid
- NATS JetStream — centrale event bus die alle detectiesilo's verbindt
- Control Daemon — consumeert NATS-events, per-peer threat scoring, quarantaine via NetBird API
- Wazuh (manager + indexer + dashboard) — SIEM met NATS→Wazuh forwarder
- Redis — threat score store + sessiestatus voor control daemon
- Python DLP ICAP-server — upload-scanning op poort `192.168.122.23:1345`

**Data plane (pop01):**
- OPNsense firewall en routing
- Squid expliciete proxy (pre-auth listener `100.70.154.79:3128`)
- SSL Bump (SASE-PoC-CA)
- URL-filtering (UT1 Toulouse Remote ACL + handmatige blacklist)
- ClamAV + c-icap (malware-scanning + DLP laag 1, ICAP RESPMOD op `127.0.0.1:1344`)
- Unbound DNS-resolver (RPZ ingeschakeld via `respip`-module)
- BIND 9.20 (os-bind, niet-resolverende secundaire zone voor TSIG-geauthenticeerde zone-transfer, poort `53530`)
- Suricata IDS (vtnet0 WAN + vtnet1 LAN, PCAP-modus)
- NetBird-agent (exit node, routing peer, WPAD-listener)

---

## 5. Verkeersstromen

### 5.1 Clientverkeer HTTP/HTTPS (het hoofdinspectiepad)

```
mobile01
  │ ① DNS-opzoeking: wpad.sandbox.local → 100.70.135.241 (mgmt01)
  │   (via NetBird primaire nameserver → pop01 Unbound → RPZ-controle → opgelost)
  │
  │ ② PAC-bestand opgehaald: http://wpad.sandbox.local/wpad.dat (Caddy op mgmt01)
  │   Resultaat: PROXY 100.70.154.79:3128 voor al het externe verkeer
  │
  │ ③ Browser stuurt CONNECT naar pop01 Squid op 100.70.154.79:3128 (WireGuard-tunnel)
  │
  ▼
pop01 Squid (pre-auth listener, ssl-bump)
  │
  ├── Identity Bridge lookup: external_acl bevraagt mgmt01 met overlay-IP (%SRC)
  │     → retourneert Entra ID persona-groep (Studenten/Docenten/Admins)
  │     → persona-ACL bepaalt policybeslissing (identiteitsgebaseerde filtering)
  ├── URL-filtercontrole (UT1 / handmatige blacklist) → 403 indien geblokkeerd
  ├── ICAP REQMOD → Python DLP (mgmt01:1345) → 403 bij gevoelige data in upload
  ├── SSL Bump: TLS-terminatie + herencryptie (SASE-PoC-CA)
  └── ICAP RESPMOD → ClamAV/c-icap (pop01:1344) → 403 bij malware of DLP-match
  │
  ▼
Internet (via pop01 vtnet0 WAN)
  │
  └── Suricata IDS op vtnet0: ziet herencrypteerde upstream-verbindingen
        TLS-metadata, DNS-anomalieën, C2-signatures, verdachte user-agent
```

### 5.2 DNS-resolutie (threat intelligence-pad)

```
mobile01 of dc01
  │ DNS-query (alle queries — via NetBird primaire nameserver)
  ▼
pop01 Unbound (poort 53, respip-module actief)
  │ RPZ-controle: staat het domein in threat-intel.rpz.sase?
  ├── Ja → NXDOMAIN (autoritatief, aa-vlag)
  └── Nee → doorsturen naar upstream DNS → antwoord teruggestuurd
```

### 5.3 Zone-transferketen (threat intelligence-updatepad)

```
URLhaus + ThreatFox feeds
  ↓ HTTP-ophalen (periodiek)
ioc2rpz (mgmt01 Docker, 192.168.122.23:53)
  ↓ TSIG-geauthenticeerde AXFR (tkey_rpz_transfer, hmac-sha256)
BIND 9.20 (pop01, 127.0.0.1:53530, secundaire zone)
  ↓ niet-geauthenticeerde AXFR (loopback, geen auth vereist)
Unbound RPZ (pop01, zone in-memory geladen via respip)
```

### 5.4 ZTNA-tunnelopbouw (identiteitsverificatiepad)

```
netbird up (mobile01)
  │ OIDC-stroom → Zitadel (mgmt01) → Entra ID (aplab.be tenant)
  │ Gate 1: Entra ID Conditional Access evalueert (✅ operationeel)
  │   - CA Policy 1: MFA vereist ✅
  │   - CA Policy 2: Geo-blokkering (alleen België) — Report-only
  │   - CA Policy 3: Legacy-auth geblokkeerd ✅
  │   - CA Policy 4: Risico-gebaseerde blokkering ✅
  │   - CA Policy 5: Conform apparaat vereist — Report-only
  │ OIDC-token ontvangen
  │
  │ Gate 2: Intune-apparaatconformiteit evalueert (✅ operationeel)
  │   - OS-versie, Defender AV + firewall, real-time protection
  │   - mobile01 enrolled als 2ITCSC1A-MOB-1, conform
  │ WireGuard-tunnelonderhandeling met pop01
  │ WireGuard-tunnel actief, ACL-policies toegepast
  ▼
pop01 (wt0-interface, overlay IP 100.70.154.79)
  Clientverkeer stroomt nu door inspectiepipeline (§5.1)
```

---

## 6. De vier-laags inspectiepipeline

Al het HTTP/HTTPS-verkeer van beheerde clients passeert vier afzonderlijke inspectielagen. De lagen zijn **aanvullend**, niet redundant — elke laag inspecteert een domein dat de anderen niet kunnen zien:

| Laag | Component | Locatie | Inspecteert | Protocol |
|------|-----------|---------|-------------|----------|
| 1 | Squid + URL-filtering | pop01 | URLs, categorieën, HTTPS-verbindingsopbouw | Applicatie |
| 2 | ClamAV + c-icap + YARA + SDD | pop01:1344 | Gedownloade bestandsinhoud (malware + DLP laag 1) | ICAP RESPMOD |
| 3 | Python DLP ICAP | mgmt01:1345 | Geüploade POST/PUT/PATCH-bodies (DLP laag 2) | ICAP REQMOD |
| 4 | Suricata IDS | pop01 vtnet0+vtnet1 | Netwerkstromen, TLS-metadata, DNS, ruwe pakketjes | Raw pcap |

Lagen 1–3 zijn sequentieel bij elke HTTP-transactie. Laag 4 draait parallel op ruwe netwerkinterfaces, onafhankelijk van de proxy-pipeline.

**Waarom Suricata NIET in de ICAP-pipeline zit:** Suricata is een pakketstroom-engine die TCP-streams herbouwt en patronen over verbindingen heen correleert. Een HTTP-body geleverd via ICAP verliest die context. De splitsing is architectureel fundamenteel, geen PoC-beperking. Zie [Beslissing: Suricata WAN+LAN](../decisions/suricata-wan-lan.md).

---

## 7. Componentenkaart

| Component | Pagina | Rol | Status |
|-----------|--------|-----|--------|
| Squid + WPAD/PAC + SSL Bump | [squid.md](../components/squid.md) | SWG-proxy, HTTPS-inspectie, URL-filtering | ✅ Operationeel |
| ClamAV + c-icap | [clamav-cicap.md](../components/clamav-cicap.md) | Malware-scanning + DLP laag 1 (downloads) | ✅ Operationeel |
| Python DLP ICAP | [python-dlp.md](../components/python-dlp.md) | DLP laag 2 (uploads/POST-bodies) | ✅ Operationeel |
| Suricata IDS | [suricata.md](../components/suricata.md) | Netwerklaag-bedreigingsdetectie | ✅ Operationeel |
| ioc2rpz + BIND + Unbound RPZ | [ioc2rpz.md](../components/ioc2rpz.md) | DNS threat intelligence | ✅ Operationeel |
| NetBird + Zitadel + Entra ID | [netbird.md](../components/netbird.md) | ZTNA-transportlaag | ✅ Operationeel |
| Caddy | [caddy.md](../components/caddy.md) | WPAD-server + reverse proxy | ✅ Operationeel |
| GNS3 / Topologie | [gns3.md](../components/gns3.md) | Virtualisatie-infrastructuur | ✅ Operationeel |
| VyOS | [vyos.md](../components/vyos.md) | SASE-gateway — Zero Trust Branch model (Site-LAN) | ✅ Operationeel |
| Identity Bridge | [identity-bridge.md](../components/identity-bridge.md) | NetBird overlay-IP → Entra ID groep (SWG↔ZTNA-koppellaag) | ✅ Operationeel |
| NATS JetStream | [nats-jetstream.md](../components/nats-jetstream.md) | Centrale event bus — verbindt detectiesilo's | ✅ Operationeel |
| Control Daemon | [control-daemon.md](../components/control-daemon.md) | Threat scoring + real-time quarantaine via NetBird API | ✅ Operationeel |
| Wazuh | [wazuh.md](../components/wazuh.md) | SIEM — NATS forwarder + pop01-agent | ✅ Operationeel |
| Entra ID CA + Intune | [ca-posture-hybrid.md](../decisions/ca-posture-hybrid.md) | Contextbewuste toegang (Gates 1+2) | ✅ Operationeel |
| M365 Activity API + Wazuh AR | [wazuh.md](../components/wazuh.md) | CASB Laag 2 — API-mode handhaving | ✅ Operationeel |

---

## 8. Vertrouwensgrenzen

| Grens | Wat het handhaaft | Waar |
|-------|------------------|------|
| WireGuard-tunnelopbouw | Identiteit (OIDC) + apparaatposture (Intune-conformiteit) | pop01 wt0 |
| Squid ACL | Netwerkniveau toestaan/weigeren per subnet | pop01 Squid |
| ICAP-pipeline | Inhoudsniveau toestaan/weigeren per transactie | pop01 + mgmt01 |
| ICAP fail-open | `bypass=on` — uploads passeren ongeïnspecteerd als Python DLP-container uitvalt | pop01 Squid → mgmt01 |
| NetBird ACL-policies | Welke peer welke resource kan bereiken | NetBird management |
| Unbound RPZ | DNS-niveau domein toestaan/weigeren | pop01 Unbound |
| OPNsense pf | Stateful firewall (basisbeveiliging) | pop01 |
| DC-LAN-uitgaand verkeer | Geen SWG-inspectie — gerouteerd door OPNsense, niet via proxy | pop01 vtnet1 gateway |

---

## 9. Event-driven handhavingslaag

Alle detectiecomponenten publiceren gestructureerde events naar een centrale NATS JetStream bus op mgmt01. Twee consumers verwerken deze events onafhankelijk:

```
Detectiesilo's (producers)
  ├── Suricata IDS (pop01)          → security.alert.ids
  ├── Squid proxy (pop01)           → security.alert.proxy
  ├── Python DLP (mgmt01)           → security.alert.dlp
  ├── c-icap/ClamAV (pop01)         → security.alert.malware
  ├── DNS-RPZ (pop01)               → security.alert.dns
  └── Identity Bridge (mgmt01)      → identity.login / identity.group_change
                    │
                    ▼
            NATS JetStream (mgmt01)
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
    Control Daemon         Wazuh SIEM
    (real-time)            (forensisch)
    - threat scoring       - custom rules (100500-100600)
    - quarantaine/herstel  - M365 Active Response
    - ENFORCE gate         - Discover dashboard
```

De dual-write architectuur garandeert dat detectie-events beschikbaar zijn voor zowel real-time respons (control daemon) als forensisch onderzoek (Wazuh) onafhankelijk. Geen van beide paden is afhankelijk van het andere.

---

## 10. Gate-model

Drie gates handhaven toegangscontrole op verschillende punten in de verbindingslevenscyclus:

| Gate | Technologie | Timing | Status |
|------|-------------|--------|--------|
| Gate 1 — Identiteit | Entra ID Conditional Access (5 policies) | Bij authenticatie (OIDC-login) | ✅ Operationeel |
| Gate 2 — Apparaat | Intune-apparaatconformiteit | Bij authenticatie + continu (8u-cyclus) | ✅ Operationeel (Report-only tot demo) |
| Gate 3 — Inhoud | Squid + ClamAV + Python DLP + Suricata + Unbound RPZ | Elke HTTP/DNS-aanvraag | ✅ Operationeel |

Gates zijn aanvullend, niet redundant. Gate 1 evalueert *wie* de gebruiker is (MFA, aanmeldingsrisico, geo-locatie). Gate 2 evalueert *wat* het apparaat is (OS-versie, AV-status, firewall). Gate 3 evalueert *wat* wordt overgedragen (malware, gevoelige data, geblokkeerde domeinen). Geen enkele gate kan de andere vervangen.

---

**DC-LAN-inspectiegat:** dc01 gebruikt pop01 als standaardgateway (`10.0.0.1`) voor internettoegang. Dit verkeer wordt gerouteerd door OPNsense en geïnspecteerd door Suricata op vtnet1, maar gaat **niet** via Squid/ICAP — er is geen WPAD/PAC- of proxyconfiguratie op dc01. DNS-queries van dc01 gaan wel via Unbound RPZ. Dit is een geaccepteerde scopebeperking: DC-resources zijn serverworkloads, geen BYOD-browsers.

---

## Gerelateerd

- [Concept: SASE](../concepts/sase.md)
- [Concept: Zero Trust / Drie-gate model](../concepts/zero-trust.md)
- [Concept: ICAP](../concepts/icap.md)
- [Concept: SSL Bump](../concepts/ssl-bump.md)
- [Concept: WPAD/PAC](../concepts/wpad-pac.md)
- [Concept: RPZ / DNS Threat Intelligence](../concepts/rpz.md)
- [Concept: DLP](../concepts/dlp.md)

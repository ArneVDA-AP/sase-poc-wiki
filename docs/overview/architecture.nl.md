---
title: "Systeemarchitectuur"
tags: [sase, architecture, network, opnsense, netbird, squid, suricata, zero-trust]
---

# Systeemarchitectuur

Deze pagina beschrijft de volledige architectuur van de SASE PoC: nodes, netwerksegmenten, de control/data plane-splitsing en hoe alle beveiligingscomponenten met elkaar samenhangen. Elke componentpagina verwijst hiernaartoe voor architecturale context.

---

## 1. Ontwerpfilosofie: Castle & Moat → Edge-handhaving

Het project verwerpt het traditionele perimetermodel waarbij vertrouwen voortkomt uit netwerkpositie. In het traditionele model beschermt een firewall één grens; iedereen binnenin is vertrouwd. Dit werkt niet voor Atlascollege: 4 000 studenten-BYOD-laptops die thuis, in cafés, treinen werken — er is geen "binnenste" meer.

In plaats daarvan vinden inspectie en vertrouwensbeslissingen plaats **op de PoP** (Point of Presence) — de edge-node tussen clients en internet. Vertrouwen wordt bepaald door drie onafhankelijk geëvalueerde factoren:

1. **Identiteit** — wie ben je? (Entra ID + MFA)
2. **Apparaatposture** — wat is de toestand van je apparaat? (NetBird posture checks)
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
| **sitepc01** | Ubuntu 24.04 (geen OS) | Gepland remote site-endpoint — nog niet operationeel | `172.16.10.50` | — |
| **mobile01** | Windows 11 (VMware, buiten GNS3) | BYOD-client — simuleert studentlaptop van elke locatie | extern | `100.70.95.98` |

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

### 5.1 BYOD HTTP/HTTPS-verkeer (het hoofdinspectiepad)

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
  │ [GEPLAND] Gate 1: Entra ID Conditional Access evalueert
  │   - MFA vereist
  │   - Geo-blokkering (alleen België)
  │   - Legacy-authenticatie geblokkeerd
  │   - Aanmeldrisico (Entra ID Protection, A5-licentie)
  │ OIDC-token ontvangen
  │
  │ WireGuard-tunnelonderhandeling met pop01
  │ [GEPLAND] Gate 2: NetBird posture checks evalueren
  │   - OS-kernel ≥ 10.0.19041
  │   - NetBird-client ≥ minimumversie
  │   - MsMpEng.exe actief (Windows Defender)
  │   - Geo: België
  │ WireGuard-tunnel actief, ACL-policies toegepast
  ▼
pop01 (wt0-interface, overlay IP 100.70.154.79)
  BYOD-verkeer stroomt nu door inspectiepipeline (§5.1)
```

---

## 6. De vier-laags inspectiepipeline

Al het HTTP/HTTPS-verkeer van BYOD-clients passeert vier afzonderlijke inspectielagen. De lagen zijn **aanvullend**, niet redundant — elke laag inspecteert een domein dat de anderen niet kunnen zien:

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
| VyOS | [vyos.md](../components/vyos.md) | SD-WAN / SASE-gateway (remote site) | ⚠️ Gedeeltelijk |
| Entra ID CA + Posture Checks | [netbird.md](../components/netbird.md) | Contextbewuste toegang (Gates 1+2) | 🔲 Gepland |

---

## 8. Vertrouwensgrenzen

| Grens | Wat het handhaaft | Waar |
|-------|------------------|------|
| WireGuard-tunnelopbouw | Identiteit (OIDC) + apparaatposture (gepland) | pop01 wt0 |
| Squid ACL | Netwerkniveau toestaan/weigeren per subnet | pop01 Squid |
| ICAP-pipeline | Inhoudsniveau toestaan/weigeren per transactie | pop01 + mgmt01 |
| NetBird ACL-policies | Welke peer welke resource kan bereiken | NetBird management |
| Unbound RPZ | DNS-niveau domein toestaan/weigeren | pop01 Unbound |
| OPNsense pf | Stateful firewall (basisbeveiliging) | pop01 |

---

## Gerelateerd

- [Concept: SASE](../concepts/sase.md)
- [Concept: Zero Trust / Drie-gate model](../concepts/zero-trust.md)
- [Concept: ICAP](../concepts/icap.md)
- [Concept: SSL Bump](../concepts/ssl-bump.md)
- [Concept: WPAD/PAC](../concepts/wpad-pac.md)
- [Concept: RPZ / DNS Threat Intelligence](../concepts/rpz.md)
- [Concept: DLP](../concepts/dlp.md)

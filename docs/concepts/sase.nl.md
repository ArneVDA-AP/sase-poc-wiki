---
title: "Concept: SASE — Secure Access Service Edge"
tags: [sase, zero-trust, network, architecture]
---

# Concept: SASE — Secure Access Service Edge

**Definitie:** Een framework dat netwerktransport (WAN Edge) en beveiligingsdiensten (SSE) samenvoegt in één cloud-native stack die op de edge wordt geleverd — waar de gebruiker is — in plaats van in een centraal datacenter.

## Hoe het hier van toepassing is

Dit project implementeert een SASE-proof-of-concept voor Atlascollege: 4000+ BYOD-studenten die verbinding maken vanaf willekeurige locaties, met Microsoft 365 als kern-applicatieplatform. Het traditionele perimetermodel (firewall + VPN = vertrouwd) werkt niet onder deze omstandigheden — studenten bevinden zich nooit "binnen de muren."

De PoC vervangt het perimetermodel door een SASE-stack waarbij:
- **Transport** een WireGuard-overlay (NetBird) is in plaats van een campus-VPN
- **Identiteit** Entra ID (Microsoft) is in plaats van netwerklocatie
- **Inspectie** plaatsvindt op pop01 (de PoP/edge-node) bij elke aanvraag, niet aan een datacentervergrenzing

**Control plane / Data plane-splitsing** — een bewuste architecturale keuze:
- **mgmt01** = management plane: distribueert configuratie (WPAD), beheert identiteit (Zitadel/Entra ID), aggregeert threat intelligence (ioc2rpz), draait Docker-services
- **pop01** = data plane: inspecteert al het verkeer (Squid, ClamAV, Suricata), handhaaft DNS-beleid (Unbound RPZ), termineert WireGuard-tunnels (wt0)

## Waar het in de stack voorkomt

| SASE-pijler | Commercieel equivalent | Onze implementatie | Status |
|-------------|------------------------|-------------------|--------|
| **ZTNA** | Zscaler ZPA | NetBird + Zitadel + Entra ID (drie-gate model) | ✅ Operationeel |
| **SWG** | Zscaler ZIA | Squid + SSL Bump + ClamAV + Python DLP + Unbound RPZ | ✅ Operationeel |
| **CASB (inline)** | Netskope CASB | Squid + ICAP DLP-pipeline (URL-filtering + DLP — gedeeltelijk; geen app-niveau controles of shadow IT-detectie) | ⚠️ Gedeeltelijk |
| **CASB (API)** | Defender for Cloud Apps | Wazuh + Microsoft Graph (gepland) | ⏳ Gepland |
| **FWaaS** | Zscaler Cloud Firewall | OPNsense + Suricata IDS | Gedeeltelijk |
| **SD-WAN** | Zscaler Zero Trust SD-WAN | VyOS site01 + NetBird op sitepc01 | Gedeeltelijk |

## Belangrijke onderscheidingen

**SASE vs SSE:** SSE (Security Service Edge) is de beveiligingsgerichte subset van SASE — ZTNA + SWG + CASB. SASE voegt de WAN Edge-component (SD-WAN) toe. Dit project implementeert SSE volledig en SD-WAN gedeeltelijk.

**PoC vs productie:** Commerciële SASE gebruikt single-pass inspectiemotoren — TLS-decodering, URL-filtering, DLP en malwarescanning in één doorgang. Deze stack is een seriële keten (Squid decodeert → ICAP naar Python DLP → ICAP naar ClamAV → herencrypteert). Inspectiebereik is equivalent; architectuur is geoptimaliseerd voor duidelijkheid boven doorvoer.

**Identiteit vs netwerkpositie:** De fundamentele verschuiving is dat toegangsbeslissingen gebaseerd zijn op *wie de gebruiker is* en *welk apparaat ze gebruiken*, niet *waar ze zich op het netwerk bevinden*. NetBird ACL-policies, Entra ID CA en de ICAP-inspectiepipeline handhaven dit gezamenlijk.

## Bronnen

- `raw/SASE_Architectuur_Overzicht.md` §1–3 (conceptueel kader, vijf pijlers, drie-gate model)
- `raw/Doc1_Squid_WPAD_PAC.md` §1 (SWG-positionering)
- `raw/Doc6_NetBird_ZTNA.md` §1 (ZTNA-positionering)

---
title: "Concept: Zero Trust"
tags: [zero-trust, sase, network, architecture]
---

# Concept: Zero Trust

**Definitie:** Een beveiligingsmodel waarbij geen enkele gebruiker of apparaat standaard wordt vertrouwd, ongeacht netwerklocatie — toegang vereist continue verificatie van identiteit, apparaatstatus en aanvraaginhoud bij elke stap.

## Hoe het hier van toepassing is

Traditionele VPN verleent netwerktoegang — eenmaal geauthenticeerd kunnen gebruikers alle resources bereiken achter de tunnel. Zero Trust vervangt dit door drie orthogonale verificatielagen, elk werkend op een ander punt in de verbindingslevenscyclus:

| Gate | Timing | Technologie | Wat het blokkeert |
|------|--------|-------------|-------------------|
| **Gate 1** | Authenticatie (OIDC-login) | Entra ID Conditional Access (5 policies) | Gestolen inloggegevens, afwijkend aanmeldingsrisico, legacy clients, niet-toegestane geografieën |
| **Gate 2** | Authenticatie + continu (8u) | Intune-apparaatconformiteit | Verouderd OS, geen antivirus, geen firewall, niet-conform apparaat |
| **Gate 3** | Elke HTTP/DNS-aanvraag | SWG-pipeline (Squid + ClamAV + DLP + Unbound RPZ) | Malware, data-exfiltratie, geblokkeerde domeinen, bekende kwaadaardige IOC's |

Alle drie de gates zijn operationeel. Gates 1 en 2 zijn geactiveerd in V40 (CA-policies) en V40 (Intune-conformiteit). Sommige policies staan op Report-only in afwachting van demo-voorbereiding (Sessie 11).

**Waarom gates aanvullend zijn, niet redundant:** Gate 1 (CA) dwingt MFA af en evalueert aanmeldingsrisico — maar identiteit alleen zegt niets over de beveiligingsstatus van het apparaat. Gate 2 (Intune-conformiteit) attesteert OS-versie, antivirus en firewall — maar kan gestolen-inloggegevenrisico niet evalueren of MFA afdwingen. Geen enkele gate kan de andere vervangen. Gate 3 vangt bedreigingen in de inhoud op die beide gates omzeilen.

## Waar het in de stack voorkomt

- **[NetBird](../components/netbird.md)** — implementeert Zero Trust Network Access: per-peer identiteit via OIDC, cryptografische peer-sleutels, granulaire ACL-policies per resourcegroep. Een gebruiker met toegang tot dc01 kan mgmt01 niet bereiken zonder een apart beleid.
- **[Entra ID CA / Drie-gate model](../decisions/ca-posture-hybrid.md)** — Gate 1 (identiteitsverificatie) en Gate 2 (apparaatposture) handhaven Zero Trust-principes voordat een tunnel tot stand wordt gebracht.
- **[Squid](../components/squid.md)** + **[ClamAV](../components/clamav-cicap.md)** + **[Python DLP](../components/python-dlp.md)** — Gate 3: assume-breach-principe, inspecteert al het verkeer ongeacht wie gates 1 en 2 heeft gepasseerd.
- **[Suricata](../components/suricata.md)** — detectie van lateral movement op vtnet1 (LAN), het "assume breach"-principe belichamend zelfs voor intern verkeer.
- **[ioc2rpz/Unbound RPZ](../components/ioc2rpz.md)** — DNS-niveau handhaving: bekende kwaadaardige domeinen worden geblokkeerd voordat een TCP-verbinding wordt opgezet.

## Belangrijke onderscheidingen

**Zero Trust vs VPN:** VPN geeft netwerktoegang; Zero Trust geeft resourcetoegang. NetBird implementeert dit via WireGuard-mesh + ACL-policies — elke resource heeft zijn eigen expliciet beleid, en peers die geen beleid hebben kunnen niet eens routeren naar die resource.

**Zero Trust-principes (Microsoft-framework) — expliciete gate-mapping:**
- *Verify Explicitly (identiteit)* → Gate 1: Entra ID CA evalueert wie de gebruiker is (MFA, aanmeldingsrisico, geolocatie)
- *Verify Explicitly (apparaat)* → Gate 2: Intune-apparaatconformiteit attesteert wat het apparaat is (OS-versie, Defender AV, firewall); NetBird posture checks blijven optionele defense-in-depth
- *Use Least Privilege* → NetBird ACL-policies per resourcegroep
- *Assume Breach* → Gate 3: SWG-pipeline + Suricata inspecteren alle inhoud in de data plane, ongeacht wie Gates 1 en 2 heeft gepasseerd

**Beheerde apparaten scope:** Na een scopewijziging (lector-mandaat, 21 april 2026) richt het project zich op beheerde Windows-apparaten (Intune-beheerd, Entra joined) in plaats van BYOD. Dit maakt het mogelijk dat Gate 2 Intune-apparaatconformiteit gebruikt voor attestatie-gebaseerde posturecontroles (OS-versie, Defender AV + firewall, real-time protection). NetBird posture checks blijven als optionele verdediging in de diepte.

## Gate-status

| Gate | Technologie | Status |
|------|-------------|--------|
| Gate 1 — Identiteit | Entra ID Conditional Access (5 policies) | ✅ Operationeel |
| Gate 2 — Apparaat | Intune-apparaatconformiteit | ✅ Operationeel (Report-only tot demo) |
| Gate 3 — Inhoud | Squid + ClamAV + Python DLP + Suricata + Unbound RPZ | ✅ Operationeel |

Zie: [Beslissing: CA + Posture hybride](../decisions/ca-posture-hybrid.md)

## Bronnen

- `raw/Doc7_ZTNA_Context_Aware.md` §1–3 (drie-gate model, CA vs posture, implementatieplan)
- `raw/SASE_Architectuur_Overzicht.md` §3 (Zero Trust-hoofdstuk)
- `raw/Doc6_NetBird_ZTNA.md` §1 (Zero Trust network access)

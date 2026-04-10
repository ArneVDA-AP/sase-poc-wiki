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
| **Gate 1** | Authenticatie (OIDC-login) | Entra ID Conditional Access | Gestolen inloggegevens, afwijkend aanmeldingsrisico, legacy clients, niet-toegestane geografieën |
| **Gate 2** | Tunnelestablishment (WireGuard) | NetBird Posture Checks | Verouderd OS, geen antivirus, niet-conforme NetBird-clientversie |
| **Gate 3** | Elke HTTP/DNS-aanvraag | SWG-pipeline (Squid + ClamAV + DLP + Unbound RPZ) | Malware, data-exfiltratie, geblokkeerde domeinen, bekende kwaadaardige IOC's |

Gates 1 en 2 zijn architecturaal ontworpen en gedocumenteerd (Addendum E, april 2026) maar nog niet geactiveerd in de sandbox. Gate 3 is volledig operationeel.

**Waarom gates aanvullend zijn, niet redundant:** Gate 1 (CA) kan MFA en aanmeldingsrisico afdwingen — maar kan apparaatstatus niet controleren op onbeheerde BYOD zonder Intune-inschrijving. Gate 2 (posture) kan OS-versie en AV-status verifiëren — maar kan gestolen-inloggegevenrisico niet evalueren of MFA afdwingen. Geen enkele gate kan de andere vervangen. Gate 3 vangt bedreigingen op die beide gates omzeilen.

## Waar het in de stack voorkomt

- **[NetBird](../components/netbird.md)** — implementeert Zero Trust Network Access: per-peer identiteit via OIDC, cryptografische peer-sleutels, granulaire ACL-policies per resourcegroep. Een gebruiker met toegang tot dc01 kan mgmt01 niet bereiken zonder een apart beleid.
- **[Entra ID CA / Drie-gate model](../decisions/ca-posture-hybrid.md)** — Gate 1 (identiteitsverificatie) en Gate 2 (apparaatposture) handhaven Zero Trust-principes voordat een tunnel tot stand wordt gebracht.
- **[Squid](../components/squid.md)** + **[ClamAV](../components/clamav-cicap.md)** + **[Python DLP](../components/python-dlp.md)** — Gate 3: assume-breach-principe, inspecteert al het verkeer ongeacht wie gates 1 en 2 heeft gepasseerd.
- **[Suricata](../components/suricata.md)** — detectie van laterale beweging op vtnet1 (LAN), het "assume breach"-principe belichamend zelfs voor intern verkeer.
- **[ioc2rpz/Unbound RPZ](../components/ioc2rpz.md)** — DNS-niveau handhaving: bekende kwaadaardige domeinen worden geblokkeerd voordat een TCP-verbinding wordt opgezet.

## Belangrijke onderscheidingen

**Zero Trust vs VPN:** VPN geeft netwerktoegang; Zero Trust geeft resourcetoegang. NetBird implementeert dit via WireGuard-mesh + ACL-policies — elke resource heeft zijn eigen expliciet beleid, en peers die geen beleid hebben kunnen niet eens routeren naar die resource.

**Zero Trust-principes (Microsoft-framework):**
- *Verify Explicitly* → Gate 1 (identiteit) + Gate 2 (apparaat)
- *Use Least Privilege* → NetBird ACL-policies per resourcegroep
- *Assume Breach* → Gate 3-pipeline + Suricata op LAN-segment

**CA-apparaatconformiteit op onbeheerde BYOD:** Entra ID CA kan apparaatconformiteit (OS-versie, AV-status, schijfversleuteling) alleen controleren voor Intune-ingeschreven apparaten. BYOD-studenten schrijven persoonlijke laptops niet in bij MDM. NetBird posture checks vullen dit gat — ze evalueren apparaatstatus bij WireGuard-tunnelbouw zonder MDM-inschrijving te vereisen.

## Bronnen

- `raw/Doc7_ZTNA_Context_Aware.md` §1–3 (drie-gate model, CA vs posture, implementatieplan)
- `raw/SASE_Architectuur_Overzicht.md` §3 (Zero Trust-hoofdstuk)
- `raw/Doc6_NetBird_ZTNA.md` §1 (Zero Trust network access)

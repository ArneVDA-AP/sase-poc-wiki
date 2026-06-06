---
title: "Concept: Application Gateway"
tags: [application-gateway, cosmos, reverse-proxy, mfa, smartshield, zero-trust, ztna, sase]
---

# Concept: Application Gateway

**Definitie:** Een identity-aware reverse proxy die voor interne applicaties staat en elke applicatie gate't op gebruikersidentiteit (login + MFA), sessiegeldigheid en bescherming per route. Het beantwoordt *welke gebruiker deze applicatie mag openen*, los van *welk device het netwerk mag betreden*.

## Hoe het hier van toepassing is

In dit project is de application gateway de vierde en bovenste laag van de SASE-stack, geïmplementeerd door [Cosmos](../components/cosmos.nl.md) op dc01. De drie lagen eronder beantwoorden andere vragen: NetBird ZTNA beantwoordt *welk device het DC-netwerk mag betreden*, OPNsense + Suricata doen L3/L4-filtering, en Squid + ClamAV inspecteren uitgaande content. Geen van die lagen beslist of een bepaalde **gebruiker** een bepaalde **applicatie** mag openen. Dat is het gat dat de application gateway vult.

De gateway dwingt dit af door de enige ingress naar de backend-services te zijn. Elke DC-service (Uptime Kuma, Gitea) is enkel bereikbaar via Cosmos op poort 80/443; de backend-container-poorten (`3001`, `3000`) zijn nooit extern gebonden. Een verzoek naar een beveiligde route wordt onderschept, de gebruiker authenticeert met login + TOTP, en pas dan wordt het verzoek naar de backend geforward. Vier eigenschappen maken dit een *application gateway* in plaats van een gewone reverse proxy:

- **Identity gate:** login + MFA per beveiligde route, niet alleen transport.
- **Session management als access control:** de Cosmos-sessie is tijdsgebonden (48u in deze PoC). Bij verlopen sluit de applicatie, ook al zit het device nog op het netwerk.
- **SmartShield:** rate-limiting, bot-detectie en anti-DDoS per route. Omdat clients via de NetBird-overlay binnenkomen, zijn de source IP's die SmartShield ziet overlay-adressen, dus internet-scanners bereiken het nooit.
- **Container-isolatie:** elke applicatie draait in zijn eigen Docker-netwerk, zodat een compromittering van één backend niet lateraal naar een andere kan bewegen.

De gateway is de tweede van twee Zero Trust-gates: device admission (NetBird) en daarna application admission (Cosmos). Zie [Beslissing: Twee-laags ZTNA](../decisions/cosmos-two-layer-ztna.nl.md) en [Concept: Zero Trust](zero-trust.nl.md).

## Waar het in de stack voorkomt

- **[Cosmos](../components/cosmos.nl.md):** de enige application gateway in dit project. Reverse-proxiet Uptime Kuma (`kuma.dc.local`, MFA-gate aan) en Gitea (`gitea.dc.local`, MFA-gate bewust uit) op dc01; past SmartShield per route toe en isoleert elke backend in zijn eigen Docker-netwerk.
- **[NetBird](../components/netbird.nl.md):** de laag *onder* de gateway. Het is geen application gateway (het laat devices toe tot het netwerk); het is de voorwaarde die de gateway bereikbaar maakt, want `10.0.0.100` en `*.dc.local` zijn enkel routeerbaar over de overlay.

## Belangrijke onderscheidingen

**Application gateway vs SWG (Secure Web Gateway).** Beide zijn proxies, maar ze kijken de tegenovergestelde richting op. Een SWG (Squid in deze stack) is een *uitgaande* gateway: het inspecteert verkeer van interne gebruikers dat *naar buiten* gaat, met URL-filtering, malware-scanning en DLP. Een application gateway is een *inkomende* gateway: het controleert verkeer van gebruikers die *naar binnen* komen naar interne applicaties, gate'end op identiteit en MFA. Cosmos is geen SWG en ziet niets van het uitgaande verkeer dat Squid afhandelt.

**Application gateway vs ZTNA network-admission.** ZTNA network-admission (NetBird) beslist welk *device* het netwerk mag betreden en het IP van een resource mag bereiken; eenmaal toegelaten kan het device elke service proberen die zijn ACL's toestaan. De application gateway voegt een tweede, user-level gate *per applicatie* toe: een device dat op de NetBird-overlay zit maar waarvan de Cosmos-sessie verlopen is, kan nog steeds geen enkele DC-applicatie openen zonder opnieuw met MFA te authenticeren. Network admission is device-scoped en eenmalig per verbinding; application admission is user-scoped, per applicatie, en wordt opnieuw gecontroleerd bij sessieverloop.

**Granulariteit per applicatie.** De gateway kan een andere posture per service toepassen. Uptime Kuma draagt de MFA-gate (operationeel gevoelige monitoring-data); Gitea niet (het heeft zijn eigen volwassen auth, dus een tweede MFA-laag zou redundant zijn). Deze keuze per service is een eigenschap van het application-gateway-model, geen beperking. Zie [Beslissing: Twee-laags ZTNA](../decisions/cosmos-two-layer-ztna.nl.md).

## Bronnen

- `rayan_Cosmos Implementatie SASE PoC.md` §1, §12 (vier-lagen-model, identity gate, SmartShield, bewuste architectuurkeuzes)
- `DEMO_SCRIPT_COSMOS_MFA.md` (MFA-gate-flow, twee-gate-framing)
- `SASE_PoC_Rayan_State_Snapshot_Juni2026.md` §2 ZTNA (Cosmos als per-applicatie toegangscontrole, Gate 2)

---
title: "Beslissing: Twee-laags Zero Trust (NetBird + Cosmos)"
tags: [decision, cosmos, application-gateway, mfa, zero-trust, ztna, netbird, sase]
---

# Beslissing: Twee-laags Zero Trust (NetBird + Cosmos)

**Status:** Geïmplementeerd (PoC-gevalideerd op de parallelle stack, sandbox-integratie pending)
**Datum:** Mei 2026 (Cosmos-implementatie, dc01)

> **Scope.** Deze beslissing is gebouwd en gevalideerd op de parallelle `SASE_POC`-omgeving (dc01 `10.0.0.100`), nog niet in het sandbox-geheel geïntegreerd. Ze is additief op de bestaande NetBird ZTNA van de sandbox, geen vervanging.

## Context

NetBird ZTNA laat een device toe tot het DC-netwerk en laat het het IP van een resource bereiken. Maar network admission is device-scoped: zodra een device op de overlay zit, kan iedereen die het gebruikt `10.0.0.100` proberen en bereiken wat de ACL's toestaan. De interne DC-services (Uptime Kuma, Gitea, toekomstige services) hadden een tweede control nodig die een andere vraag beantwoordt, namelijk *welke gebruiker deze specifieke applicatie mag openen*, met sterke authenticatie, en idealiter met granulariteit per applicatie zodat een gevoelig dashboard en een developer-tool niet dezelfde posture hoeven te dragen.

De vraag was of device-level ZTNA op zich volstaat, of dat een aparte application-admission-laag gerechtvaardigd is, en zo ja, welke technologie die moet leveren.

## Overwogen opties

| Optie | Voor | Tegen |
|-------|------|-------|
| Enkel NetBird (device admission) | Eén laag om te beheren; geen extra component | Geen user-level, per-applicatie-gate; een gedeeld/onvergrendeld device op de overlay bereikt elke service; geen MFA op de applicatie |
| Cosmos Constellation VPN (ingebouwd) | Eén vendor voor netwerk + app | Betaalde functie in v0.22.10; dupliceert NetBird's overlay; zou twee overlays betekenen |
| Enkel app-native auth (bv. Gitea's eigen 2FA) | Geen proxy-laag; per definitie per applicatie | Inconsistent over services heen; services zonder sterke auth blijven blootgesteld; geen centrale sessie-/MFA-gate; geen container-isolatie |
| **NetBird (netwerk) + Cosmos (applicatie)** | Twee orthogonale gates; user-level MFA per app; granulariteit per service; container-isolatie + SmartShield als bonus | Extra component op dc01; Cosmos hostname-op-IP beperkt cross-domain OAuth |

## Beslissing

Implementeer Zero Trust als **twee lagen**: NetBird voor network admission (Gate 1, device-level) en [Cosmos](../components/cosmos.nl.md) als [application gateway](../concepts/application-gateway.nl.md) voor application admission (Gate 2, user-level met MFA).

```
GATE 1 — NetBird:  welk device mag het DC-netwerk betreden?   (device-level)
GATE 2 — Cosmos:   welke gebruiker mag deze applicatie openen? (user-level, login + MFA via TOTP)
```

Cosmos Constellation VPN wordt expliciet niet gebruikt (betaald, en het zou NetBird's overlay dupliceren); NetBird adverteert `10.0.0.0/8` aan alle peers, wat dc01 routeerbaar maakt, dus Cosmos hoeft enkel applicaties te gate'en, niet het transport.

Binnen Gate 2 wordt de posture **per service** gezet:

- **Uptime Kuma, MFA-gate aan.** Het monitoring-dashboard toont operationeel gevoelige staat over de hele SASE-stack, dus het krijgt de maximale stack: NetBird + Cosmos MFA + Kuma's eigen login.
- **Gitea, MFA-gate uit (bewust).** Gitea heeft volwassen native auth (SSH keys, PAT's, optionele 2FA) en developers loggen meerdere keren per dag in; een tweede Cosmos MFA op dezelfde identiteit zou redundant zijn. De toegangscontrole zit hier in NetBird (device) + Gitea's eigen auth (user). Een technisch detail versterkt de keuze: de `AuthEnabled`-toggle is grijs voor `*.dc.local`-routes door de hostname-op-IP-beperking, maar de architecturale motivatie staat los van die beperking. Zie [Bevinding: hostname op IP beperkt OAuth](../findings/cosmos-hostname-oauth.nl.md).

## Gevolgen

- De twee gates zijn orthogonaal en aanvullend: Gate 1 kan geen per-applicatie gebruikersidentiteit afdwingen, en Gate 2 kan niet bestaan zonder het netwerkpad dat Gate 1 levert. Geen van beide vervangt de andere.
- Een device op de NetBird-overlay waarvan de Cosmos-sessie verlopen is (48u timeout in deze PoC) kan geen enkele MFA-gated DC-applicatie openen zonder opnieuw met MFA te authenticeren. Sessieverloop wordt een access-control-event, niet enkel een UX-detail.
- De application-gateway-laag brengt twee extra voordelen gratis mee: SmartShield rate-limiting per route, en Docker-container-isolatie (elke backend in zijn eigen netwerk, poorten niet extern gebonden), wat lateral movement beperkt als één backend gecompromitteerd raakt.
- Granulariteit per service is nu een eersteklas mogelijkheid: de admin bepaalt de posture per applicatie, gedemonstreerd door de Kuma-aan / Gitea-uit-splitsing.
- De prijs is één extra component op dc01 en de hostname-op-IP-beperking (grijze `AuthEnabled` op `*.dc.local`, geen cross-domain OAuth/SSO). Cosmos migreren naar `cosmos.dc.local` zou die beperking opheffen; uitgesteld. Zie [Bevinding: hostname op IP beperkt OAuth](../findings/cosmos-hostname-oauth.nl.md).
- Constellation VPN blijft uit; als er later een behoefte aan een Cosmos-native overlay ontstaat, moeten de betaalde-functie- en dubbele-overlay-afwegingen opnieuw bekeken worden.

Zie ook: [Component: Cosmos](../components/cosmos.nl.md), [Component: NetBird](../components/netbird.nl.md), [Concept: Application Gateway](../concepts/application-gateway.nl.md), [Concept: Zero Trust](../concepts/zero-trust.nl.md), [Runbook 13: Cosmos](../runbooks/13-cosmos.nl.md)

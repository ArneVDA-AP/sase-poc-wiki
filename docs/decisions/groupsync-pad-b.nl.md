---
title: "Beslissing: GroupSync Pad B — Zitadel verwijdert Entra ID-prefix"
tags: [decision, groupsync, zitadel, entra-id, netbird]
---

# Beslissing: GroupSync Pad B — Zitadel verwijdert Entra ID-prefix

**Status:** Geïmplementeerd  
**Datum:** Mei 2026 (Verslag30)

## Context

Entra ID-groepsnamen dragen een verplicht `2ITcsc1A-`-prefix opgelegd door een lector-mandaat (de tenant wordt gedeeld over meerdere studentenprojecten). Zo verschijnt de groep voor studenten als `2ITcsc1A-Studenten` in Entra ID. Dit prefix moet ergens in de synchronisatiepipeline worden afgehandeld, omdat NetBird-toegangspolicies en Squid ACL's werken op groepsnamen — policies die verwijzen naar `2ITcsc1A-Studenten` in plaats van `Studenten` zijn moeilijker leesbaar, moeilijker te onderhouden, en propageren het prefix door elk downstream-component (Identity Bridge, NATS-subjects, Squid external ACL-responses).

## Overwogen opties

| Optie | Voor | Tegen |
|-------|------|-------|
| **Pad A: Prefix overal meenemen** | Geen transformatielogica nodig; namen zijn 1:1 met Entra ID | Elke policy, ACL en elk NATS-subject moet de volledige prefixed naam gebruiken (`2ITcsc1A-Studenten`). Vermindert leesbaarheid. Prefix lekt door naar Identity Bridge-mapping, NATS-subjecthiërarchie en Squid ACL-output |
| **Pad B: Zitadel verwijdert prefix** | Schone interne namen (`Studenten`). Policies en ACL's zijn leesbaar. Geen prefix-propagatie door de stack | Vereist een Zitadel Action om Entra ID GUID naar schone naam te mappen. Mismatch tussen Zitadel-output en NetBird JWT-sync kan lockout veroorzaken |

## Beslissing

Pad B — een Zitadel Action mapt de Entra ID-groeps-GUID naar een schone interne naam, waarbij het `2ITcsc1A-`-prefix wordt verwijderd. Het prefix bestaat alleen in Entra ID; alle downstream-componenten (Zitadel-claims, NetBird JWT auto-groups, Identity Bridge, NATS-subjects, Squid ACL's) gebruiken de schone naam.

De primaire reden is leesbaarheid van policies: wanneer een NetBird ACL of Squid-regel verwijst naar `Studenten`, is het onmiddellijk duidelijk welke groep wordt beheerd. Bij `2ITcsc1A-Studenten` moet elke lezer het prefix mentaal verwijderen om de beleidsintentie te begrijpen.

## Gevolgen

- De Zitadel Action moet een allowlist-mapping onderhouden van Entra ID-groeps-GUID's naar schone namen. Het toevoegen van een nieuwe groep in Entra ID vereist het bijwerken van deze mapping.
- Een mismatch tussen de outputgroepsnaam van Zitadel en wat NetBird verwacht in JWT-claims resulteert erin dat peers niet aan de juiste auto-groups worden toegewezen — feitelijk een lockout-scenario.
- Identity Bridge hoeft geen prefix-stripping uit te voeren; het ontvangt reeds schone namen uit het door Zitadel uitgegeven JWT.
- NATS-subjects gebruiken schone groepsnamen (bijv. `groupsync.Studenten`), wat de subjecthiërarchie leesbaar houdt.
- Als het lector-mandaat de prefixconventie wijzigt, hoeft alleen de Zitadel Action-mapping te worden bijgewerkt — geen downstream-wijzigingen nodig.

Zie ook: [Component: Zitadel](../components/zitadel.md), [Component: Identity Bridge](../components/identity-bridge.md), [Component: NetBird](../components/netbird.md), [Concept: Identity Flow](../concepts/identity-flow.md)

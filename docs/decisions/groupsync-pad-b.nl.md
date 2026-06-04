---
title: "Beslissing: GroupSync Pad B — Zitadel verwijdert Entra ID-prefix"
tags: [decision, groupsync, zitadel, entra-id, netbird]
---

# Beslissing: GroupSync Pad B — Zitadel verwijdert Entra ID-prefix

**Status:** Geïmplementeerd  
**Datum:** Mei 2026 (Verslag30–34; Pad B bevestigd voor alle drie persona's in Verslag34)

## Context

Entra ID-groepsnamen dragen een verplicht `2ITCSC1A-`-prefix opgelegd door een lector-mandaat (de tenant wordt gedeeld over meerdere studentenprojecten). Zo verschijnt de groep voor studenten als `2ITCSC1A-Studenten` in Entra ID. Dit prefix moet ergens in de synchronisatiepipeline worden afgehandeld, omdat NetBird-toegangspolicies en Squid ACL's werken op groepsnamen — policies die verwijzen naar `2ITCSC1A-Studenten` in plaats van `Studenten` zijn moeilijker leesbaar, moeilijker te onderhouden, en propageren het prefix door elk downstream-component (Identity Bridge, NATS-subjects, Squid external ACL-responses).

## Overwogen opties

| Optie | Voor | Tegen |
|-------|------|-------|
| **Pad A: Prefix overal meenemen** | Geen transformatielogica nodig; namen zijn 1:1 met Entra ID | Elke policy, ACL en elk NATS-subject moet de volledige prefixed naam gebruiken (`2ITCSC1A-Studenten`). Vermindert leesbaarheid. Prefix lekt door naar Identity Bridge-mapping, NATS-subjecthiërarchie en Squid ACL-output |
| **Pad B: Zitadel verwijdert prefix** | Schone interne namen (`Studenten`). Policies en ACL's zijn leesbaar. Geen prefix-propagatie door de stack | Vereist dat de bestaande Zitadel Action de groeps-displaynaam-string via een allowlist naar de schone naam mapt. Harde voorwaarde: `cloud_displayname` moet werken, zodat de JWT-claim de naam `2ITCSC1A-Studenten` draagt in plaats van een ObjectID-GUID (een GUID staat niet in de allowlist en kan niet naar `Studenten` gemapt worden). Een mismatch tussen de gemapte naam en wat NetBird/Squid verwachten veroorzaakt een lockout |

## Beslissing

Pad B — de Zitadel Action die al in de OIDC-keten zit mapt de groeps-displaynaam-string naar de schone interne naam (`2ITCSC1A-Studenten` → `Studenten`) via een hardgecodeerde **allowlist**, en laat elke groep die niet in de allowlist staat vallen (fail-closed). Het netto-effect is dat het `2ITCSC1A-`-prefix NetBird nooit bereikt. Dit hangt ervan af dat `cloud_displayname` is ingeschakeld in het Entra-app-manifest, zodat de `groups`-claim de displaynaam uitstuurt in plaats van de GUID; die voorwaarde is geverifieerd, en Pad B is bevestigd werkend voor alle drie persona's (Studenten en Admins per Verslag31, Docenten per Verslag34). Het prefix bestaat alleen in Entra ID; alle downstream-componenten (Zitadel-claims, NetBird JWT auto-groups, Identity Bridge, NATS-subjects, Squid ACL's) gebruiken de schone naam.

De primaire reden is leesbaarheid van policies: wanneer een NetBird ACL of Squid-regel verwijst naar `Studenten`, is het onmiddellijk duidelijk welke groep wordt beheerd. Bij `2ITCSC1A-Studenten` moet elke lezer het prefix mentaal verwijderen om de beleidsintentie te begrijpen. Pad B laat ook Addendum J's `NETBIRD_POLICY_GROUPS`-config (`Studenten,Docenten,Admins`) al kloppend en hergebruikt een bestaand transformatiepunt in plaats van er een toe te voegen.

## Gevolgen

- De correctheid van Pad B hangt af van `cloud_displayname`: als dit ooit stopt met het uitsturen van displaynamen (bijv. omdat de groepen als on-prem-gesynchroniseerd worden behandeld), valt de claim terug op GUID's en matcht de allowlist (gekeyd op de naamstring) niets — elke groep wordt fail-closed weggelaten, en de fragiele fallback zou GUID-matching in ACL's of een GUID→naam-lookup zijn. Dit wordt geverifieerd in de GroupSync `cloud_displayname`-stap vóór men op Pad B vertrouwt.
- De schone interne naam moet **byte-voor-byte** overeenkomen over de Identity Bridge external ACL, Addendum J's `NETBIRD_POLICY_GROUPS` en elke NetBird ACL-policy. Een mismatch (hoofdletters, spaties) blokkeert alle betrokken peers met 401 op elke API-request — inclusief de admin — en herstel vereist directe SQLite-interventie op mgmt01 (GitHub #5399).
- Identity Bridge hoeft geen prefix-stripping uit te voeren; het ziet de reeds schone groepsnaam (`Studenten`) op de peer wanneer het de NetBird-API pollt.
- Als het lector-mandaat de prefixconventie wijzigt, hoeft alleen de strip-logica van de Zitadel Action te worden bijgewerkt — geen downstream-wijzigingen nodig.

Zie ook: [Component: Zitadel](../components/zitadel.md), [Component: Identity Bridge](../components/identity-bridge.md), [Component: NetBird](../components/netbird.md), [Concept: Identity Flow](../concepts/identity-flow.md)

---
title: "Concept: Identity Flow — Entra ID tot Squid"
tags: [identity, entra-id, zitadel, netbird, squid, identity-bridge, zero-trust]
---

# Concept: Identity Flow — Entra ID tot Squid

**Definitie:** De volledige identiteitspropagatie-keten die het Entra ID-groepslidmaatschap van een gebruiker transporteert vanuit de Microsoft-clouddirectory, door vier tussenliggende systemen, tot aan de per-request toegangsbeslissingen van de Squid-proxy.

## Hoe het hier van toepassing is

In een commercieel SASE-product (Zscaler, Netskope) stuurt de agent identiteitsheaders mee met elk verzoek — identiteitspropagatie is een ingebouwde functie van de client-naar-cloud-tunnel. In deze PoC bestrijkt geen enkel component het volledige pad van authenticatie tot per-request filtering. In plaats daarvan verloopt identiteitspropagatie via een multi-hop-keten waarbij elk systeem groepslidmaatschap doorgeeft aan het volgende:

```
Entra ID (aplab.be)
  | OIDC-token met groups claim (GUID's) + cloud_displayname optional claim
  v
Zitadel (mgmt01 Docker)
  | Action 1: GUID -> clean naam (verwijdert 2ITcsc1A-prefix)
  | Action 2: setClaim('groups', [...]) in JWT
  v
NetBird Management (mgmt01 Docker)
  | JWT group sync: leest groups claim, maakt/wijst auto_groups toe
  | Persona-groepen: Studenten, Docenten, Admins
  v
Identity Bridge (mgmt01 Docker, FastAPI)
  | Pollt NetBird API elke 30s: GET /api/peers + GET /api/users
  | Bouwt in-memory cache: {overlay_ip -> {email, groups[], os}}
  v
Squid (pop01, external_acl)
  | Per verzoek: helper bevraagt Identity Bridge met %SRC (overlay-IP)
  | Retourneert persona-groep -> ACL evalueert per-groep-beleid
  v
Per-request identiteitsgebaseerde filtering
  (bijv. Studenten geblokkeerd voor ChatGPT; Docenten toegestaan)
```

Elke hop transformeert de identiteitsrepresentatie: GUID's worden clean namen, clean namen worden JWT-claims, JWT-claims worden auto-groups, auto-groups worden gecachte IP-naar-groep-mappings, en gecachte mappings worden ACL-beslissingen. Een fout op eender welke hop verbreekt de keten — maar het fail-open ontwerp zorgt ervoor dat een gebroken identiteitsketen nooit authenticatie blokkeert, alleen de toegang beperkt tot het meest restrictieve standaardbeleid.

## GroupSync Path B

De Entra ID `groups`-claim bevat GUID's die betekenisloos zijn voor NetBird. Er bestaan twee paden om deze naar bruikbare groepsnamen te converteren:

**Path A — IdP Sync via API:** NetBird pollt Microsoft Graph API rechtstreeks om groepslidmaatschappen op te lossen. Dit vereist een commerciele NetBird-licentie en is niet beschikbaar in de Community Edition die in deze PoC wordt gebruikt.

**Path B — JWT group sync (geimplementeerd):** Zitadel Action 1 leest de `cloud_displayname` optional claim uit het Entra ID-token. Deze claim draagt de leesbare groepsnaam met het tenant-teamprefix (`2ITcsc1A-Studenten`). De action verwijdert het prefix en produceert clean namen (`Studenten`, `Docenten`, `Admins`). Action 2 injecteert deze clean namen vervolgens in de JWT die aan NetBird wordt uitgegeven.

Path B heeft een fundamentele beperking: groepslidmaatschap wordt alleen bijgewerkt wanneer de gebruiker inlogt. Er is geen achtergrondsynchronisatie. Als een gebruiker aan een nieuwe Entra ID-groep wordt toegevoegd, is de wijziging pas zichtbaar in de stack bij het volgende OIDC-authenticatie-event van die gebruiker.

Zie [Beslissing: GroupSync Path B](../decisions/groupsync-pad-b.md).

## Propagatievertraging

Wijzigingen in Entra ID-groepslidmaatschap hebben tijd nodig om door de volledige keten te propageren. Elke hop introduceert een eigen vertraging:

| Hop | Vertraging | Trigger |
|-----|------------|---------|
| Entra ID -> Zitadel | Volgende gebruikerslogin | Geen achtergrondsync (Path B-beperking) |
| Zitadel -> NetBird | Zelfde login-event | JWT group sync is synchroon |
| NetBird -> Identity Bridge | Maximaal 30s | Polling-interval |
| Identity Bridge -> Squid | Volgend verzoek | external_acl TTL (30s cache, 10s negatief) |

**Totaal slechtste geval:** volgende login + 30s + volgend verzoek. In de praktijk betekent dit dat een wijziging in groepslidmaatschap vereist dat de gebruiker opnieuw authenticeert bij NetBird voordat de nieuwe groep effect heeft in het filterbeleid van Squid. Voor beveiligingskritieke groepsverwijderingen (bijv. het intrekken van admin-toegang) is deze vertraging significant.

## Fail-open ontwerp

De identiteitsketen is ontworpen om op elke hop fail-open te zijn in plaats van toegang te blokkeren:

- **Zitadel Actions:** `allowed-to-fail: true` — authenticatie slaagt zonder group claims
- **Identity Bridge:** retourneert "unknown" voor niet-oplosbare IP's — Squid past standaard restrictief beleid toe
- **Squid external_acl:** valt terug op niet-identiteitsgebaseerde URL-filtering wanneer Identity Bridge onbereikbaar is

Dit is bewust: Gate 3 (SWG-pijplijn — inhoudsinspectie, malwarescanning, DLP) werkt onafhankelijk van identiteit. Een gebruiker zonder group claims krijgt nog steeds volledige inhoudsinspectie; deze heeft alleen geen toegang tot groepsspecifiek beleid (bijv. alleen-Docenten URL-toelatingen). Beveiliging is gelaagd, niet afhankelijk van een enkele identiteitsketen.

## Waar het in de stack voorkomt

- **[Zitadel](../components/zitadel.md)** — OIDC-broker, Action 1 (GUID-naar-naam-mapping) en Action 2 (JWT group injection)
- **[NetBird](../components/netbird.md)** — JWT group sync leest de `groups`-claim en maakt auto_groups per persona aan
- **[Identity Bridge](../components/identity-bridge.md)** — pollt NetBird API, bouwt overlay IP-naar-groep-cache, bedient Squid-lookups
- **[Squid](../components/squid.md)** — external_acl bevraagt Identity Bridge per verzoek, dwingt persona-gebaseerd filterbeleid af
- **[Entra ID CA](../decisions/ca-posture-hybrid.md)** — Gate 1 Conditional Access-beleid geevalueerd tijdens de OIDC-stroom

## Belangrijke onderscheidingen

**Identiteit vs authenticatie:** Authenticatie (bewijzen wie je bent) vindt eenmalig plaats tijdens de OIDC-loginstroom via Zitadel en Entra ID. Identiteitspropagatie (dat bewijs door de stack transporteren zodat downstream-systemen toegangsbeslissingen kunnen nemen) vindt continu plaats via de polling- en cachingketen. Een gebruiker kan geauthenticeerd zijn zonder dat de identiteit gepropageerd is — dit is de fail-open toestand.

**Path A vs Path B:** Path A (IdP Sync) pollt Graph API rechtstreeks en werkt groepslidmaatschap op de achtergrond bij — het werkt zelfs wanneer gebruikers niet actief inloggen. Path B (JWT group sync) is afhankelijk van het OIDC-token dat bij het inloggen wordt uitgegeven — groepswijzigingen propageren pas wanneer de gebruiker opnieuw authenticeert. Deze PoC gebruikt Path B omdat dit beschikbaar is in NetBird Community Edition.

**Fail-open vs fail-closed:** De identiteitsketen faalt open by design. Dit contrasteert met de inhoudsinspectieketen (Gate 3), waar ClamAV-scanning niet wordt omzeild wanneer de service uitvalt. De rationale: identiteitscontroles bepalen *welk* beleid van toepassing is, terwijl inhoudsinspectie baseline beveiliging biedt voor *al het* verkeer ongeacht identiteit.

## Bronnen

- `raw/18_mei_SASE_PoC_Addendum_H_Identity_Bridge_v1.md` (Identity Bridge-architectuur)
- `raw/18_mei_SASE_PoC_Addendum_GroupSync.md` (JWT group sync, Path B)
- `raw/GroupSync_Correcties_Sessie1(1).md` (Zitadel Actions-implementatie)

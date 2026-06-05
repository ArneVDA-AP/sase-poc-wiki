---
title: "Concept: Identity Flow (Entra ID tot Squid)"
tags: [identity, entra-id, zitadel, netbird, squid, identity-bridge, zero-trust]
---

# Concept: Identity Flow (Entra ID tot Squid)

**Definitie:** De keten die Entra ID-groepslidmaatschap van een gebruiker propageert van de Microsoft-clouddirectory, via vier tussensystemen, naar de per-request toegangsbeslissingen in Squid.

## Hoe het hier van toepassing is

In een commercieel SASE-product (Zscaler, Netskope) stuurt de agent identiteitsheaders mee met elk verzoek; identiteitspropagatie is een ingebouwde functie van de client-naar-cloud-tunnel. In deze PoC bestrijkt geen enkel component het volledige pad van authenticatie tot per-request filtering. Identiteitspropagatie verloopt daarom via een multi-hop-keten waarbij elk systeem groepslidmaatschap doorgeeft aan het volgende:

```
Entra ID (aplab.be)
  | OIDC-token: groups claim stuurt weergavenamen uit (2ITCSC1A-Studenten) via cloud_displayname
  v
Zitadel (mgmt01 Docker)
  | Action 1: allowlist-mapt weergavenaam -> schone naam (2ITCSC1A-Studenten -> Studenten)
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
  | Per verzoek: helper queryet Identity Bridge met %SRC (overlay-IP)
  | Retourneert persona-groep -> ACL evalueert per-groep policies
  v
Per-request identiteitsgebaseerde filtering
  (bijv. Studenten geblokkeerd voor deepai.org; Docenten toegestaan)
```

Elke hop transformeert de identiteitsrepresentatie: geprefixte weergavenamen worden schone namen, schone namen worden JWT-claims, JWT-claims worden auto-groups, auto-groups worden gecachte IP-naar-groep-mappings, en gecachte mappings worden ACL-beslissingen. Een fout op eender welke hop verbreekt de keten, maar het fail-open ontwerp zorgt ervoor dat een gebroken identiteitsketen nooit authenticatie blokkeert, alleen de toegang beperkt tot de meest restrictieve default policy.

## GroupSync: sync-mechanisme en Pad B

**Sync-mechanisme (JWT group sync).** NetBird Community Edition kan de IdP Sync- (Graph API-polling) of SCIM-provisioning-mechanismen niet gebruiken (beide vereisen een NetBird Cloud- of Commercial License). Het enige mechanisme dat in CE beschikbaar is, is **JWT group sync**. NetBird leest de `groups`-claim uit het ID-token bij elke gebruikerslogin en maakt/wijst dienovereenkomstig auto-groups toe. De inherente beperking is dat groepslidmaatschap alleen bij login wordt bijgewerkt; er is geen achtergrondsynchronisatie. Als een gebruiker aan een nieuwe Entra ID-groep wordt toegevoegd, is de wijziging pas zichtbaar in de stack bij het volgende OIDC-authenticatie-event van die gebruiker.

**Prefixafhandeling, Pad B (geïmplementeerd).** Entra ID-groepsnamen dragen een verplicht `2ITCSC1A-`-teamprefix (bijv. `2ITCSC1A-Studenten`). Omdat `cloud_displayname` aan de groups claim is gekoppeld (zie [Runbook 08](../runbooks/08-groupsync.md)), stuurt de claim de *weergavenaam-string* van elke groep uit in plaats van zijn ObjectID-GUID. Er werden twee paden overwogen:

- **Pad A (prefix overal meenemen):** de volledige naam `2ITCSC1A-Studenten` propageert ongewijzigd naar NetBird, de Identity Bridge ACL en Addendum J's `NETBIRD_POLICY_GROUPS`. Traceerbaar maar verbose, en het prefix cascadeert door elke downstream-config.
- **Pad B (Zitadel strijkt het prefix weg, geïmplementeerd):** Action 1 allowlist-mapt de naamstring `2ITCSC1A-Studenten` → `Studenten` voordat NetBird die ziet; Action 2 injecteert de schone namen (`Studenten`, `Docenten`, `Admins`) in de JWT. Interne configs blijven leesbaar en Addendum J's bestaande `Studenten,Docenten,Admins` klopt al.

De harde voorwaarde voor Pad B is dat `cloud_displayname` werkt; een GUID staat niet in de allowlist en kan niet naar `Studenten` resolven. Die voorwaarde is geverifieerd, en Pad B is bevestigd werkend voor alle drie persona's.

Zie [Beslissing: GroupSync Pad B](../decisions/groupsync-pad-b.md).

## Propagatievertraging

Wijzigingen in Entra ID-groepslidmaatschap hebben tijd nodig om door de volledige keten te propageren. Elke hop introduceert een eigen vertraging:

| Hop | Vertraging | Trigger |
|-----|------------|---------|
| Entra ID -> Zitadel | Volgende gebruikerslogin | Geen achtergrondsync (JWT group sync-beperking) |
| Zitadel -> NetBird | Zelfde login-event | JWT group sync is synchroon |
| NetBird -> Identity Bridge | Maximaal 30s | Polling-interval |
| Identity Bridge -> Squid | Volgend verzoek | external_acl TTL (30s cache, 10s negatief) |

**Totaal slechtste geval:** volgende login + 30s + volgend verzoek. In de praktijk betekent dit dat een wijziging in groepslidmaatschap vereist dat de gebruiker opnieuw authenticeert bij NetBird voordat de nieuwe groep effect heeft in de filtering policies van Squid. Voor beveiligingskritieke groepsverwijderingen (bijv. het intrekken van admin-toegang) is deze vertraging significant.

## Fail-open ontwerp

De identiteitsketen is ontworpen om op elke hop fail-open te zijn in plaats van toegang te blokkeren:

- **Zitadel Actions:** `allowed-to-fail: true`, authenticatie slaagt zonder group claims
- **Identity Bridge:** retourneert "unknown" voor niet-oplosbare IP's; Squid past standaard een restrictieve policy toe
- **Squid external_acl:** valt terug op niet-identiteitsgebaseerde URL-filtering wanneer Identity Bridge onbereikbaar is

Dit is bewust: Gate 3 (SWG-pijplijn: inhoudsinspectie, malwarescanning, DLP) werkt onafhankelijk van identiteit. Een gebruiker zonder group claims krijgt nog steeds volledige inhoudsinspectie; deze heeft alleen geen toegang tot groepsspecifieke policies (bijv. alleen-Docenten URL-toelatingen). Beveiliging is gelaagd, niet afhankelijk van een enkele identiteitsketen.

## Waar het in de stack voorkomt

- **[Zitadel](../components/zitadel.md):** OIDC-broker, Action 1 (allowlist-mapt de groeps-weergavenaam-string naar de schone naam) en Action 2 (JWT group injection)
- **[NetBird](../components/netbird.md):** JWT group sync leest de `groups`-claim en maakt auto_groups per persona aan
- **[Identity Bridge](../components/identity-bridge.md):** pollt NetBird API, bouwt overlay IP-naar-groep-cache, bedient Squid-lookups
- **[Squid](../components/squid.md):** external_acl queryet Identity Bridge per verzoek, dwingt persona-gebaseerde filtering policies af
- **[Entra ID CA](../decisions/ca-posture-hybrid.md):** Gate 1 Conditional Access policies geevalueerd tijdens de OIDC-stroom

## Belangrijke onderscheidingen

**Identiteit vs authenticatie:** Authenticatie (bewijzen wie je bent) vindt eenmalig plaats tijdens de OIDC-loginstroom via Zitadel en Entra ID. Identiteitspropagatie (dat bewijs door de stack transporteren zodat downstream-systemen toegangsbeslissingen kunnen nemen) vindt continu plaats via de polling- en cachingketen. Een gebruiker kan geauthenticeerd zijn zonder dat de identiteit gepropageerd is; dit is de fail-open toestand.

**JWT group sync vs IdP Sync:** IdP Sync (en SCIM) pollen of pushen groepslidmaatschap op de achtergrond (ze werken bij zelfs wanneer gebruikers niet actief inloggen), maar beide vereisen een NetBird Cloud- of Commercial License, dus geen van beide is beschikbaar in de hier gebruikte Community Edition. JWT group sync, het enige in CE beschikbare mechanisme, is afhankelijk van het OIDC-token dat bij het inloggen wordt uitgegeven, dus groepswijzigingen propageren pas wanneer de gebruiker opnieuw authenticeert. Daarnaast verwijst **Pad A vs Pad B** naar prefixafhandeling *binnen* JWT group sync (Pad B strijkt het `2ITCSC1A-`-prefix weg, zie boven), niet naar de keuze van sync-mechanisme.

**Fail-open vs fail-closed:** De identiteitsketen faalt open by design. Dit contrasteert met de inhoudsinspectieketen (Gate 3), waar ClamAV-scanning niet wordt omzeild wanneer de service uitvalt. De rationale: identiteitscontroles bepalen *welke* policy van toepassing is, terwijl inhoudsinspectie baseline beveiliging biedt voor *al het* verkeer ongeacht identiteit.

## Bronnen

- `18_mei_SASE_PoC_Addendum_H_Identity_Bridge_v1.md` (Identity Bridge-architectuur)
- `18_mei_SASE_PoC_Addendum_GroupSync.md` (JWT group sync, Path B)
- `GroupSync_Correcties_Sessie1(1).md` (Zitadel Actions-implementatie)

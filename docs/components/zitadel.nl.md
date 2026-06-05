---
title: "Zitadel: OIDC Identity Provider Broker"
tags: [zitadel, oidc, identity, entra-id, netbird, ztna]
---

# Zitadel: OIDC Identity Provider Broker

**Rol:** OIDC Identity Provider broker die tussen NetBird en Entra ID zit en Microsoft-identiteiten vertaalt naar JWT-claims die NetBird kan gebruiken voor groepsgebaseerde toegangscontrole.  
**Versie:** Gedeployed via NetBird quickstart Docker Compose op mgmt01  
**Configuratielocatie:** Docker Compose-stack op mgmt01 (onderdeel van NetBird-implementatie)

---

## Hoe het werkt in deze stack

Zitadel is de OIDC IdP waartegen NetBird authenticeert. Het werd geinstalleerd als onderdeel van het NetBird quickstart-script, dat Zitadel als primaire OIDC-uitgever instelt in plaats van NetBird rechtstreeks aan een externe IdP te koppelen. Entra ID (aplab.be-tenant) is geconfigureerd als externe IdP *binnen Zitadel*, waardoor de volledige authenticatiestroom is:

```
NetBird-client
  -> Zitadel (OIDC-uitgever op mgmt01)
    -> Entra ID (aplab.be-tenant, externe IdP)
      -> Microsoft-inlogpagina
    <- OIDC-token: groups claim = weergavenaam-strings (via cloud_displayname)
  <- JWT met clean groups claim
-> NetBird Management leest groups uit JWT
```

Zitadel voegt een centrale gebruikersbeheerlaag toe met rollen en groepen die onafhankelijk zijn van Entra ID-configuratie. Dit is architectureel significant: het ontkoppelt het identiteitsmodel van de ZTNA-overlay van de Microsoft-directory, waardoor groepsnaamgeving en -mapping lokaal beheerd kunnen worden.

---

## Zitadel Actions

Twee aangepaste Actions vormen de kritieke brug tussen de ruwe claims van Entra ID en de clean groepsnamen die NetBird nodig heeft. Beide zijn JavaScript-snippets die server-side door Zitadel worden uitgevoerd tijdens de authenticatiestroom.

### Action 1: External Authentication (Pre-creation hook)

Wanneer een gebruiker voor het eerst via Entra ID inlogt, draagt de `groups`-claim van het Entra ID-token de groepslidmaatschappen van de gebruiker. Omdat `cloud_displayname` in het app-manifest aan de `groups`-claim is gekoppeld (zie [Runbook 08](../runbooks/08-groupsync.md)), stuurt de claim de **weergavenaam-string** van elke groep uit (bijv. `2ITCSC1A-Studenten`) in plaats van zijn ObjectID-GUID (bijv. `e5f8a2b3-...`).

Action 1 (`mapEntraGroupsToMetadata`) produceert de schone interne namen via een **allowlist** gekeyd op die weergavenaam-strings:

1. Lees de `groups`-claim (weergavenaam-strings)
2. Zoek elke naam op in een hardgecodeerde allowlist: `2ITCSC1A-Studenten` → `Studenten`, `2ITCSC1A-Docenten` → `Docenten`, `2ITCSC1A-Admins` → `Admins` (dit is [GroupSync Path B](../decisions/groupsync-pad-b.md))
3. Schrijf de overeenkomende schone namen naar de gebruikersmetadata-sleutel `sase_groups` (komma-gescheiden)

De lookup is gekeyd op de **naamstring**, en daarom is `cloud_displayname` een harde voorwaarde: een ruwe GUID kan niet ge-allowlist worden naar `Studenten`. De allowlist is bewust **fail-closed**: een groep die niet in de map staat wordt stilzwijgend weggelaten in plaats van doorgegeven. Dit is strikter dan een blinde prefix-strip. De schone namen houden downstream NetBird-ACL's en Squid-policies leesbaar.

### Action 2: Complement Token

Voegt de `groups`-claim toe aan het JWT-token dat Zitadel aan NetBird uitgeeft. Zonder deze action bevat de JWT wel authenticatie-informatie, maar geen groepslidmaatschap, en ziet NetBird nooit tot welke persona-groep de gebruiker behoort.

De action leest `sase_groups` uit de metadata van de gebruiker en roept `setClaim('groups', [...])` aan om die schone namen in de uitgaande JWT te injecteren. De JWT group sync van NetBird leest vervolgens deze claim en maakt of wijst auto-groups dienovereenkomstig toe.

### Fail-open ontwerp

Beide actions hebben `allowed-to-fail: true`. Als een van beide actions faalt, slaagt de authenticatie alsnog: de gebruiker krijgt een geldig JWT maar zonder group claims. Dit is een bewuste ontwerpkeuze. Fail-open op de authenticatielaag is acceptabel omdat Gate 3 (de SWG-pijplijn op pop01) inhoudsinspectie afdwingt ongeacht groepslidmaatschap. Een gebruiker zonder group claims ontvangt het meest restrictieve standaardbeleid in plaats van volledig buitengesloten te worden.

---

## Configuratie

### App Registration (Entra ID-zijde)

De Entra ID-app-registratie waarnaar Zitadel federeert is sandbox-specifiek:

| Eigenschap | Waarde |
|------------|--------|
| App-naam | `2ITCSC1A-Netbird-Sandbox` |
| Client ID | `11803ee8-eb15-462c-a286-5415c17a29c6` |
| Tenant ID | `23e9bcdc-5cb9-4867-9310-76cc0b462ddc` |

Deze vervangt de gedeelde registratie `cebe0d74-be9f-49ac-9f35-65f11586c1bb` die door andere teams werd gebruikt. De sandbox-specifieke registratie voorkomt onderlinge interferentie met Conditional Access-beleid en tokenconfiguratie.

### Token Configuration (Entra ID-zijde)

De `cloud_displayname`-eigenschap moet in het manifest van de Entra ID-app-registratie aan de `groups`-claim gekoppeld worden. Het is niet selecteerbaar in de Token Configuration UI en moet worden toegevoegd door het manifest rechtstreeks te bewerken (zie [Runbook 08](../runbooks/08-groupsync.md)). Zonder deze koppeling valt de `groups`-claim terug op GUID's, heeft Action 1's allowlist geen naamstring om op te matchen (een GUID staat nooit in de allowlist), en faalt groepsresolutie stilzwijgend (vanwege `allowed-to-fail: true`).

---

## Integratiepunten

| Component | Richting | Wat |
|-----------|----------|-----|
| [NetBird](netbird.md) | <- OIDC | Zitadel is de geconfigureerde OIDC-uitgever van NetBird; JWT group sync is afhankelijk van Action 2 |
| Entra ID (aplab.be) | -> federatie | Externe IdP die Microsoft-identiteiten en groepslidmaatschap levert |
| [Identity Bridge](identity-bridge.md) | indirect | Groepen propageren via de Zitadel -> NetBird -> Identity Bridge polling-keten |
| [Squid](squid.md) | indirect | Eindconsument van identiteit: persona-groepen bepalen per-request filterbeleid |

---

## Bekende problemen / valkuilen

**Geen stdout/console-logging in Actions.** Zitadel Actions hebben geen mechanisme voor debug-uitvoer. Er is geen mogelijkheid om tussenliggende waarden te loggen of uitvoering te traceren binnen een action-script. De enige verificatiemethode is het onderzoeken van JWT-claims in NetBird na een geslaagde login, wat iteratieve ontwikkeling traag en foutgevoelig maakt.

**GitHub #5399 (embedded-Dex scope-drop, niet van toepassing op deze stack).** Upstream #5399 meldt dat op NetBird-builds met *embedded Dex* + Zitadel de JWT group sync niet out-of-the-box werkt omdat `AUTH_SUPPORTED_SCOPES` de groups-scope mist. Deze sandbox draait de **oudere multi-container NetBird-stack zonder Dex-container** (Management v0.67.0 valideert Zitadel's token rechtstreeks, Verslag30), dus de door de action geïnjecteerde `groups`-claim stroomt zonder scope-fix door. De scope-drop-variant van #5399 is hier dus niet van toepassing; er is geen Docker Compose-aanpassing nodig voor group sync. (Het aparte gevaar "gevuld JWT allow-groups → 401-lockout", soms ook onder #5399 bijgehouden, geldt nog steeds; zie hieronder en [NetBird](netbird.nl.md).)

**`cloud_displayname`-afhankelijkheid:** Als `cloud_displayname` niet in het app-manifest aan de `groups`-claim is gekoppeld, draagt de claim GUID's in plaats van namen en matcht Action 1's allowlist niets, waardoor elke groep stilzwijgend wordt weggelaten. De gebruiker authenticeert succesvol maar zonder groepsresolutie. Omdat `allowed-to-fail: true` de fout onderdrukt, is deze faalwijze onzichtbaar zonder downstream JWT-claims te controleren.

**Action-uitvoering is ondoorzichtig.** Er is geen dashboard-weergave die de uitvoeringsgeschiedenis of foutaantallen van actions toont. Debugging vereist end-to-end testen: inloggen als testgebruiker en vervolgens de resulterende JWT in NetBird inspecteren om te verifieren dat group claims aanwezig en correct gemapt zijn.

---

## Gerelateerd

- [NetBird](netbird.md)
- [Identity Bridge](identity-bridge.md)
- [Concept: Identity Flow](../concepts/identity-flow.md)
- [Beslissing: Zitadel als IdP-broker](../decisions/zitadel-idp-broker.md)
- [Beslissing: GroupSync Path B](../decisions/groupsync-pad-b.md)
- [Runbook: ZTNA-overlay](../runbooks/02-ztna-overlay.md)

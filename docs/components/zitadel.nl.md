---
title: "Zitadel — OIDC Identity Provider Broker"
tags: [zitadel, oidc, identity, entra-id, netbird, ztna]
---

# Zitadel — OIDC Identity Provider Broker

**Rol:** OIDC Identity Provider broker — zit tussen NetBird en Entra ID en vertaalt Microsoft-identiteiten naar JWT-claims die NetBird kan gebruiken voor groepsgebaseerde toegangscontrole.  
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
    <- OIDC-token met groups (GUID's) + cloud_displayname
  <- JWT met clean groups claim
-> NetBird Management leest groups uit JWT
```

Zitadel voegt een centrale gebruikersbeheerlaag toe met rollen en groepen die onafhankelijk zijn van Entra ID-configuratie. Dit is architectureel significant: het ontkoppelt het identiteitsmodel van de ZTNA-overlay van de Microsoft-directory, waardoor groepsnaamgeving en -mapping lokaal beheerd kunnen worden.

---

## Zitadel Actions

Twee aangepaste Actions vormen de kritieke brug tussen de ruwe claims van Entra ID en de clean groepsnamen die NetBird nodig heeft. Beide zijn JavaScript-snippets die server-side door Zitadel worden uitgevoerd tijdens de authenticatiestroom.

### Action 1 — External Authentication (Pre-creation hook)

Wanneer een gebruiker voor het eerst via Entra ID inlogt, bevat het Entra ID-token twee relevante claims:

- `groups` — een array van GUID's (bijv. `e5f8a2b3-...`) die Azure AD-groepslidmaatschap identificeren
- `cloud_displayname` — een optional claim met de leesbare groepsnaam inclusief het tenant-teamprefix (bijv. `2ITcsc1A-Studenten`)

Action 1 voert de GUID-naar-naam-mapping uit door:

1. `cloud_displayname` uit het Entra ID-token te lezen
2. Het `2ITcsc1A-`-prefix te verwijderen (dit is [GroupSync Path B](../decisions/groupsync-pad-b.md))
3. De interne groepsnaam in te stellen op de clean versie: `Studenten`, `Docenten`, `Admins`

Deze mappingstap is noodzakelijk omdat NetBird GUID's niet kan interpreteren — het heeft leesbare groepsnamen nodig om auto-groups aan te maken.

### Action 2 — Complement Token

Voegt de `groups`-claim toe aan het JWT-token dat Zitadel aan NetBird uitgeeft. Zonder deze action bevat de JWT wel authenticatie-informatie maar geen groepslidmaatschap, en ziet NetBird nooit tot welke persona-groep de gebruiker behoort.

De action roept `setClaim('groups', [...])` aan om de gemapte groepsnamen in de uitgaande JWT te injecteren. De JWT group sync van NetBird leest vervolgens deze claim en maakt of wijst auto-groups dienovereenkomstig toe.

### Fail-open ontwerp

Beide actions hebben `allowed-to-fail: true`. Als een van beide actions faalt, slaagt de authenticatie alsnog — de gebruiker krijgt een geldig JWT maar zonder group claims. Dit is een bewuste ontwerpkeuze: fail-open op de authenticatielaag is acceptabel omdat Gate 3 (de SWG-pijplijn op pop01) inhoudsinspectie afdwingt ongeacht groepslidmaatschap. Een gebruiker zonder group claims ontvangt het meest restrictieve standaardbeleid in plaats van volledig buitengesloten te worden.

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

De `cloud_displayname` optional claim moet expliciet geconfigureerd worden in de Entra ID-app-registratie onder Token Configuration. Zonder deze claim heeft Action 1 geen leesbare groepsnaam om mee te werken en faalt de GUID-naar-naam-mapping stilzwijgend (vanwege `allowed-to-fail: true`).

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

**Geen stdout/console-logging in Actions** — Zitadel Actions hebben geen mechanisme voor debug-uitvoer. Er is geen mogelijkheid om tussenliggende waarden te loggen of uitvoering te traceren binnen een action-script. De enige verificatiemethode is het onderzoeken van JWT-claims in NetBird na een geslaagde login, wat iteratieve ontwikkeling traag en foutgevoelig maakt.

**GitHub #5399: JWT group sync met embedded Dex** — JWT group sync werkt niet out-of-the-box bij gebruik van de embedded Dex + Zitadel-combinatie. De `AUTH_SUPPORTED_SCOPES`-omgevingsvariabele mist vereiste scopes. Dit vereist handmatige aanpassing van de NetBird Docker Compose-configuratie.

**`cloud_displayname`-afhankelijkheid** — Als de Entra ID Token Configuration geen `cloud_displayname` als optional claim bevat, faalt Action 1 stilzwijgend. De gebruiker authenticeert succesvol maar zonder groepsmapping. Omdat `allowed-to-fail: true` de fout onderdrukt, is deze faalwijze onzichtbaar zonder downstream JWT-claims te controleren.

**Action-uitvoering is ondoorzichtig** — Er is geen dashboard-weergave die de uitvoeringsgeschiedenis of foutaantallen van actions toont. Debugging vereist end-to-end testen: inloggen als testgebruiker en vervolgens de resulterende JWT in NetBird inspecteren om te verifieren dat group claims aanwezig en correct gemapt zijn.

---

## Gerelateerd

- [NetBird](netbird.md)
- [Identity Bridge](identity-bridge.md)
- [Concept: Identity Flow](../concepts/identity-flow.md)
- [Beslissing: Zitadel als IdP-broker](../decisions/zitadel-idp-broker.md)
- [Beslissing: GroupSync Path B](../decisions/groupsync-pad-b.md)
- [Runbook: ZTNA-overlay](../runbooks/02-ztna-overlay.md)

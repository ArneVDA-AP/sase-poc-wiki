---
title: "Runbook: GroupSync"
tags: [runbook, identity, entra-id, zitadel, netbird, groupsync]
---

# Runbook: GroupSync

**Node(s):** Entra ID (aplab.be) + Zitadel (mgmt01) + NetBird Dashboard
**Vereisten:** [Runbook 02: ZTNA Overlay](02-ztna-overlay.nl.md) afgerond (overlay operationeel), Entra ID-beheerderstoegang op aplab.be
**Status:** Operationeel

---

## Vereistenchecklist

- [ ] NetBird-overlay is operationeel ([Runbook 02](02-ztna-overlay.nl.md))
- [ ] Entra ID-beheerderstoegang op aplab.be-tenant
- [ ] Zitadel-beheerderstoegang op mgmt01
- [ ] App registration `11803ee8-eb15-462c-a286-5415c17a29c6` bestaat in Entra ID

---

## Stap 1: Token Configuration configureren in Entra ID

Navigeer in de Azure-portal naar de app registration (`11803ee8-eb15-462c-a286-5415c17a29c6`):

1. Ga naar **Token configuration** in het linkermenu
2. Klik op **Add optional claim**
3. Selecteer tokentype: **ID**
4. Vink `cloud_displayname` aan
5. Opslaan

Dit zorgt ervoor dat de weergavenaam van de gebruiker (inclusief het klasprefix) wordt opgenomen in het ID-token dat naar Zitadel wordt gestuurd.

---

## Stap 2: Groups claim configureren

Nog steeds in Token configuration:

1. Klik op **Add groups claim**
2. Selecteer: **Groups assigned to the application** (niet "All groups")
3. Zorg ervoor dat group claims zijn ingeschakeld voor elk tokentype (ID, Access, SAML)
4. Opslaan

> **Valkuil: Selecteer NIET "All groups".** Alle groepen selecteren omvat elke groep in de tenant, wat het token opblaast en JWT-groottelimieten kan bereiken. "Groups assigned to the application" retourneert alleen de 3 beveiligingsgroepen die expliciet aan deze Enterprise Application zijn toegewezen.

---

## Stap 3: Beveiligingsgroepen toewijzen aan de Enterprise Application

Navigeer naar **Enterprise Applications** → zoek de bijbehorende Enterprise Application:

1. Ga naar **Users and groups** → **Add user/group**
2. Wijs deze 3 beveiligingsgroepen toe:

| Beveiligingsgroep | Mapt naar persona | Doel |
|-------------------|-------------------|------|
| SASE-MobileUsers | Studenten | BYOD mobiele gebruikers met beperkte toegang |
| SASE-SiteUsers | Docenten | Sitegebruikers met standaardtoegang |
| SASE-Admins | Admins | Volledige beheerderstoegang |

3. Bevestig dat alle 3 groepen in de toewijzingslijst verschijnen

---

## Stap 4: Gebruikerstoewijzing vereist inschakelen

Nog steeds op de Enterprise Application:

1. Ga naar **Properties**
2. Stel **User assignment required** in op **Yes**
3. Opslaan

Dit voorkomt dat niet-toegewezen gebruikers zich authenticeren via de applicatie. Alleen gebruikers in een van de 3 beveiligingsgroepen kunnen inloggen.

---

## Stap 5: Zitadel Actions aanmaken

Navigeer in de Zitadel-beheerconsole op mgmt01 naar **Actions**:

### Action 1 — External Authentication

**Trigger:** External Authentication
**Allowed to fail:** true

Deze actie:

- Leest de `cloud_displayname` claim uit het Entra ID-token
- Verwijdert het `2ITcsc1A-`-klasprefix uit de weergavenaam
- Mapt GUID's uit de groups claim naar schone groepsnamen (Studenten, Docenten, Admins)

### Action 2 — Complement Token

**Trigger:** Complement Token
**Allowed to fail:** true

Deze actie:

- Roept `setClaim('groups', [...])` aan om de opgeloste groepsnamen in het door Zitadel uitgegeven JWT te injecteren
- NetBird leest deze `groups` claim om gebruikers aan personagroepen toe te wijzen

> **Valkuil: Beide acties moeten `allowed-to-fail: true` hebben.** Als een actie faalt (bijv. Entra ID retourneert niet de verwachte claim voor een gebruiker), moet authenticatie toch slagen — de gebruiker heeft dan alleen geen personagroepen toegewezen. Zie [Beslissing: GroupSync PAD B](../decisions/groupsync-pad-b.nl.md).

---

## Stap 6: Gebruikerslogin en groepstoewijzing verifiëren

1. Log in als testgebruiker die lid is van een van de 3 beveiligingsgroepen
2. Open het NetBird Dashboard → **Peers**
3. Zoek de peer van de testgebruiker

**Verwacht:** De peer verschijnt met de juiste personagroep (Studenten, Docenten of Admins) op basis van hun Entra ID-beveiligingsgroeplidmaatschap.

---

## Stap 7: Personagroepen in NetBird verifiëren

1. Open NetBird Dashboard → **Groups**
2. Controleer of personagroepen bestaan: Studenten, Docenten, Admins
3. Controleer of elke groep de juiste leden bevat op basis van Entra ID-toewijzingen

---

## Eindverificatie

Voer end-to-end-validatie uit:

```
netbird up  (als testgebruiker)
```

1. NetBird Dashboard → Peers → testgebruiker toont juiste personagroep
2. Identity Bridge `/lookup`-endpoint retourneert de juiste persona voor het IP van de gebruiker:

```bash
curl http://192.168.122.23:<port>/lookup?ip=<gebruiker-overlay-ip>
# Verwacht: retourneert personagroepnaam (Studenten/Docenten/Admins)
```

---

## Checklist

- [ ] `cloud_displayname` optional claim toegevoegd aan token configuration
- [ ] Groups claim ingesteld op "Groups assigned to the application"
- [ ] 3 beveiligingsgroepen toegewezen aan Enterprise Application
- [ ] User assignment required = Yes
- [ ] Zitadel Action 1 (External Authentication) aangemaakt met `allowed-to-fail: true`
- [ ] Zitadel Action 2 (Complement Token) aangemaakt met `allowed-to-fail: true`
- [ ] Testgebruiker login toont juiste personagroep op NetBird Dashboard
- [ ] Personagroepen (Studenten/Docenten/Admins) bestaan in NetBird met juiste leden

---

## Gerelateerd

- [Component: NetBird](../components/netbird.nl.md)
- [Component: Identity Bridge](../components/identity-bridge.nl.md)
- [Beslissing: GroupSync PAD B](../decisions/groupsync-pad-b.nl.md)
- [Beslissing: Zitadel als IdP Broker](../decisions/zitadel-idp-broker.nl.md)
- [Concept: Zero Trust](../concepts/zero-trust.nl.md)
- [Runbook 09: Identity Bridge](09-identity-bridge.nl.md)

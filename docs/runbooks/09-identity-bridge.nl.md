---
title: "Runbook: Identity Bridge"
tags: [runbook, identity-bridge, fastapi, squid, docker]
---

# Runbook: Identity Bridge

**Node(s):** mgmt01 (Docker — Identity Bridge-container), pop01 (Squid external ACL)
**Vereisten:** [Runbook 02: ZTNA Overlay](02-ztna-overlay.nl.md) afgerond (NetBird operationeel), [Runbook 08: GroupSync](08-groupsync.nl.md) afgerond
**Status:** Operationeel

---

## Vereistenchecklist

- [ ] NetBird-overlay operationeel ([Runbook 02](02-ztna-overlay.nl.md))
- [ ] GroupSync afgerond — personagroepen (Studenten/Docenten/Admins) gevuld in NetBird ([Runbook 08](08-groupsync.nl.md))
- [ ] Docker geïnstalleerd op mgmt01
- [ ] NetBird service user PAT beschikbaar voor API-authenticatie

---

## Stap 1: Identity Bridge-container deployen op mgmt01

De Identity Bridge is een FastAPI-applicatie die NetBird peer overlay-IP's mapt naar personagroepen door de NetBird Management API te pollen.

Bouw en start de Docker-container op mgmt01:

1. Plaats de Identity Bridge-broncode op mgmt01
2. Bouw de Docker-image
3. Configureer omgevingsvariabelen:

| Variabele | Waarde | Doel |
|-----------|--------|------|
| `NETBIRD_API_URL` | NetBird API-endpoint (localhost of Docker-netwerk) | API-basis-URL |
| `NETBIRD_PAT` | Service user Personal Access Token | API-authenticatie |
| `POLL_INTERVAL` | `30` (seconden, standaard) | Hoe vaak peer-naar-groep-mappings vernieuwd worden |

4. Start de container, stel de lookup-poort beschikbaar op `192.168.122.23`

> **Valkuil: Gebruik een service user PAT, geen persoonlijk admin-token.** De PAT moet leestoegang hebben tot peers en groups. Zie [Beslissing: NetBird Service PAT](../decisions/netbird-service-pat.nl.md) voor waarom een dedicated service user is aangemaakt.

---

## Stap 2: Controleren of Identity Bridge draait

```bash
# Controleer of container actief is
docker ps | grep identity-bridge

# Test het lookup-endpoint direct
curl http://192.168.122.23:<port>/lookup?ip=<bekend-peer-ip>
# Verwacht: JSON-antwoord met personagroepnaam
```

De Identity Bridge pollt de NetBird API elke 30 seconden en cachet de IP-naar-groep-mapping in het geheugen. Het eerste antwoord kan tot 30 seconden duren na opstarten terwijl de initiële poll wordt voltooid.

---

## Stap 3: Squid external_acl_type configureren op pop01

Configureer Squid op pop01 om de Identity Bridge te gebruiken als external ACL helper voor identiteitsgebaseerde filtering.

Bewerk de Squid-configuratie en voeg toe:

1. **External ACL type-definitie:** een helperscript dat de Identity Bridge bevraagt op `http://192.168.122.23:<port>/lookup?ip=%SRC`
2. **ACL-definities** op basis van personagroepen:

| ACL-naam | Personagroep | Toegangsniveau |
|----------|-------------|----------------|
| `studenten_acl` | Studenten | Beperkt (geblokkeerde categorieën) |
| `docenten_acl` | Docenten | Standaard (meeste sites toegestaan) |
| `admins_acl` | Admins | Volledige toegang |

3. **http_access-regels** die deze ACL's gebruiken om gedifferentieerde filtering af te dwingen

Herlaad Squid na configuratie:

```bash
squid -k reconfigure
```

---

## Stap 4: Controleren of identiteit in Squid-logs verschijnt

1. Browse vanaf mobile01 (als testgebruiker met bekende persona)
2. Controleer Squid access log op pop01:

```bash
tail -f /var/log/squid/access.log
```

**Verwacht:** Logregels bevatten de identiteitsgroep (Studenten/Docenten/Admins) voor elk verzoek, wat bewijst dat Squid de Identity Bridge succesvol bevraagt voor elke verbinding.

---

## Stap 5: Identiteitsgebaseerde filtering testen

Test of beleidshandhaving verschilt per persona:

**Student (beperkt):**

```bash
curl -x http://100.70.154.79:3128 https://chatgpt.com
# Verwacht: 403 Forbidden (geblokkeerd voor studenten)
```

**Docent (standaard):**

```bash
curl -x http://100.70.154.79:3128 https://chatgpt.com
# Verwacht: 200 OK (toegestaan voor docenten)
```

> **Opmerking:** Om verschillende persona's te testen, moet je inloggen als gebruikers die tot verschillende Entra ID-beveiligingsgroepen behoren (SASE-MobileUsers vs SASE-SiteUsers).

---

## Eindverificatie

End-to-end-validatie:

1. `netbird up` als studentgebruiker → NetBird Dashboard toont groep Studenten
2. Identity Bridge `/lookup` retourneert "Studenten" voor het overlay-IP van die gebruiker
3. `curl -x http://100.70.154.79:3128 https://chatgpt.com` → 403
4. `netbird up` als docentgebruiker → NetBird Dashboard toont groep Docenten
5. Identity Bridge `/lookup` retourneert "Docenten" voor het overlay-IP van die gebruiker
6. `curl -x http://100.70.154.79:3128 https://chatgpt.com` → 200

---

## Checklist

- [ ] Identity Bridge Docker-container draait op mgmt01
- [ ] Service user PAT geconfigureerd voor NetBird API-toegang
- [ ] Polling-interval ingesteld (standaard 30s)
- [ ] `/lookup?ip=...`-endpoint retourneert juiste persona voor bekende peers
- [ ] Squid external_acl_type geconfigureerd om Identity Bridge te bevragen
- [ ] Squid ACL's gedefinieerd voor Studenten/Docenten/Admins
- [ ] Squid access.log toont identiteitsgroep per verzoek
- [ ] Student geblokkeerd voor ChatGPT (403)
- [ ] Docent toegestaan tot ChatGPT (200)

---

## Gerelateerd

- [Component: Identity Bridge](../components/identity-bridge.nl.md)
- [Component: NetBird](../components/netbird.nl.md)
- [Component: Squid](../components/squid.nl.md)
- [Beslissing: NetBird Service PAT](../decisions/netbird-service-pat.nl.md)
- [Beslissing: GroupSync PAD B](../decisions/groupsync-pad-b.nl.md)
- [Concept: Zero Trust](../concepts/zero-trust.nl.md)
- [Runbook 08: GroupSync](08-groupsync.nl.md)

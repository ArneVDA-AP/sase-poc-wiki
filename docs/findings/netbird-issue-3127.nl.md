---
title: "Bevinding: NetBird API-polling met gebruiker-PAT verwijdert JWT auto-groups"
tags: [finding, netbird, identity-bridge, api]
---

# Bevinding: NetBird API-polling met gebruiker-PAT verwijdert JWT auto-groups

**Component:** [Identity Bridge](../components/identity-bridge.md), [NetBird](../components/netbird.md)  
**Ernst:** Blokkerend

## Wat er gebeurde

Het pollen van de NetBird Management API met een reguliere gebruiker-PAT verwijderde JWT-gepropageerde auto-groups van ALLE peers van die gebruiker bij elke API-aanroep. Groups verdwenen van peers binnen seconden na het starten van de Identity Bridge. Peer-routing en ACL-policies die afhankelijk waren van die groups braken onmiddellijk.

## Oorzaak

NetBird interpreteert een API-aanroep zonder JWT group-context als "geen groups meer aanwezig" en verwijdert gepropageerde groups van de peers van de aanroepende gebruiker. Een PAT draagt alleen de API-authenticatie-identiteit, niet de originele JWT-claims — dus de group-context ontbreekt. De management server verzoent het groepslidmaatschap van de gebruiker bij elke geauthenticeerde API-interactie, en een interactie zonder JWT groups resulteert in het verwijderen van groups.

## Oplossing / workaround

Gebruik een **service-user** met een admin-role PAT voor alle Identity Bridge API-polling. Service users hebben geen JWT groups en geen eigen peers — het verwijdereffect kan niet optreden omdat er geen gepropageerde groups zijn om te verzoenen en geen peers om te beïnvloeden.

```
# NetBird Dashboard → Users → Service Users → Create
# Role: admin
# Genereer PAT → gebruik in Identity Bridge config
```

## Lessen

- Admin-role service accounts zijn het correcte patroon voor API-polling in NetBird CE self-hosted wanneer JWT group sync actief is
- Reguliere gebruiker-PATs zijn ongeschikt voor geautomatiseerde API-toegang in omgevingen met JWT group-propagatie — het neveneffect is destructief en stil
- Gerelateerd aan NetBird issue #3127

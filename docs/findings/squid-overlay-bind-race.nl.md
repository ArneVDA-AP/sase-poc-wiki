---
title: "Bevinding: Squid kan overlay-listener niet binden wanneer NetBird wt0 niet klaar is"
tags: [finding, squid, netbird, startup]
---

# Bevinding: Squid kan overlay-listener niet binden wanneer NetBird wt0 niet klaar is

**Component:** [Squid](../components/squid.md), [NetBird](../components/netbird.md)  
**Ernst:** Valkuil

## Wat er gebeurde

Squid probeert de overlay-listener te binden op `100.x.x.x` voordat NetBird wt0 het overlay-IP heeft toegewezen. Het resultaat is errno 49 (EADDRNOTAVAIL). Squid faalt stilzwijgend — er wordt geen overlay-listener aangemaakt, maar het proces blijft draaien. Van buitenaf lijkt Squid gezond maar is het onbereikbaar op het overlay-netwerk.

Dit werd veroorzaakt door reboots, failover-tests (V43 `ifconfig vtnet0 down/up`) en GUI-regen-scenario's.

## Oorzaak

Race condition in de opstartvolgorde: Squid start voordat NetBird klaar is met de tunnelonderhandeling. De wt0-interface en het bijbehorende overlay-IP zijn nog niet beschikbaar op het moment dat Squid probeert te binden. FreeBSD rc.d-volgorde dwingt geen afhankelijkheid af op een volledig opgezette NetBird-overlay.

## Oplossing / workaround

Introduceer een opstartvertraging of expliciete rc.d-afhankelijkheidsvolgorde op pop01. Het starten van Squid moet worden geblokkeerd totdat de wt0-interface aanwezig is en een toegewezen IP-adres heeft.

Een eenvoudige controle vóór het starten van Squid:

```bash
# Wacht tot wt0 een IP heeft voordat Squid start
until ifconfig wt0 2>/dev/null | grep -q 'inet 100\.'; do
  sleep 1
done
```

## Lessen

- Elke service die bindt aan een NetBird overlay-IP moet rekening houden met het feit dat de interface nog niet klaar kan zijn — dit is een structureel probleem, geen eenmalige race
- Squid herprobeert geen mislukte binds na opstart; een mislukte bind is permanent totdat het proces wordt herstart
- De fout is stilzwijgend: Squid logt geen fout voor een bind-failure op een niet-bestaande interface, wat diagnose bemoeilijkt zonder expliciet de listenerstatus te controleren

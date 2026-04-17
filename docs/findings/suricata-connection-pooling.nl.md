---
title: "Bevinding: Suricata vuurt één keer per SID per flow (Squid-verbindingspooling)"
tags: [suricata, fwaas, squid, finding]
---

# Bevinding: Suricata vuurt één keer per SID per flow (Squid-verbindingspooling)

**Component:** [Suricata](../components/suricata.md)  
**Ernst:** Inzicht — geen bug

## Wat er gebeurde

Tijdens F9-validatietests genereerden meerdere `curl.exe`-verzoeken naar dezelfde bestemming via de Squid-proxy slechts één Suricata-alert per SID, ongeacht het aantal verzonden verzoeken. Dit leek aanvankelijk op alertonderdrukking of een Suricata-configuratieprobleem.

```powershell
# mobile01 — drie keer herhaald
curl.exe -x http://100.70.154.79:3128 http://testmyids.com/
curl.exe -x http://100.70.154.79:3128 http://testmyids.com/
curl.exe -x http://100.70.154.79:3128 http://testmyids.com/
# Resultaat: slechts één SID 2100498-alert in eve.json
```

## Oorzaak

Squid-verbindingspooling hergebruikt upstream TCP-verbindingen. Vanuit het perspectief van Suricata op vtnet0 verschijnen meerdere opeenvolgende HTTP-verzoeken naar dezelfde server als verkeer binnen één TCP-flow. Suricata vuurt correct één keer per SID per flow — dit is verwacht gedrag by design, geen onderdrukking.

Het aantal alerts in `eve.json` weerspiegelt het aantal afzonderlijke upstream TCP-flows, niet het aantal HTTP-verzoeken van de client.

## Oplossing

Geen oplossing nodig — dit is correct gedrag. Voor tests die een nieuwe Suricata-alert vereisen voor dezelfde SID:

- Gebruik een andere bestemmingshost per testrun
- Wacht tot de upstream Squid-inactieve time-out verstrijkt, waardoor de gepoolde verbinding wordt gesloten
- Test tegen afzonderlijke URL's die elk onafhankelijk de SID triggeren

## Lessen

- Houd bij het interpreteren van Suricata-alertaantallen tijdens proxy-gebaseerde tests rekening met Squid-verbindingspooling. Één alert per SID per flow is verwacht en correct.
- Dit heeft geen invloed op productiedetectie: werkelijk afzonderlijke kwaadaardige flows genereren elk hun eigen alert.
- Onderzocht en bevestigd in Verslag23 (Bevinding 23.6).

## Gerelateerd

- [Suricata](../components/suricata.md)
- [Squid](../components/squid.md)
- [Testen: Acceptatietests](../testing/acceptance-tests.md)

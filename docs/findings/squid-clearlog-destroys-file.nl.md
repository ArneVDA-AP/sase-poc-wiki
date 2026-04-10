---
title: "Bevinding: Squid WebUI 'Logboek wissen' verwijdert het logbestand"
tags: [finding, squid, workaround]
---

# Bevinding: Squid WebUI "Logboek wissen" verwijdert het logbestand

**Component:** [Squid](../components/squid.md)  
**Ernst:** Valkuil

## Wat er gebeurde

Na het klikken op "Logboek wissen" (→ "dit logboek leegmaken?") in Services → Squid Web Proxy → Diagnostics → Access Log verdween het toegangslog volledig. Volgende proxytests toonden geen logvermeldingen, wat suggereerde dat de proxy niet werkte — maar die werkte correct.

Dit leidde tot een langdurige probleemoplossingssessie voor een probleem dat niet bestond.

## Oorzaak

De WebUI-knop "Logboek wissen" verwijdert het logbestand in plaats van het af te kappen. Na verwijdering blijft Squid draaien maar schrijft naar een niet-bestaande bestandsdescriptor, wat geen loguitvoer oplevert. Het logbestand wordt pas opnieuw aangemaakt wanneer Squid herstart.

## Oplossing / workaround

Om het logboek af te kappen zonder het te verwijderen:
```bash
> /var/log/squid/access.log
```

Als het bestand al verwijderd was:
```bash
configctl proxy restart
```

Dit zorgt ervoor dat Squid het logbestand opnieuw aanmaakt bij het opstarten.

## Lessen

- Gebruik nooit de WebUI-knop "Logboek wissen" voor Squid — kapt altijd af vanuit de shell
- Wanneer proxytests geen loguitvoer tonen: controleer eerst of het logbestand bestaat (`ls -la /var/log/squid/access.log`) vóór de aanname dat de proxy kapot is
- Loggerelateerde problemen kunnen onderliggende problemen maskeren — verifieer altijd dat het logbestand bestaat vóór het starten van proxy-diagnose

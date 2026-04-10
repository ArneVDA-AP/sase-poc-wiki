---
title: "Bevinding: StevenBlack-hostsformaat incompatibel met OPNsense Remote ACL"
tags: [finding, squid, workaround]
---

# Bevinding: StevenBlack-hostsformaat incompatibel met OPNsense Remote ACL

**Component:** [Squid](../components/squid.md)  
**Ernst:** Valkuil

## Wat er gebeurde

De handleiding (v4 §27.1) suggereerde StevenBlack unified hosts als Remote ACL-bron voor URL-filtering. Na configuratie laadde OPNsense Remote ACL de lijst niet en toonde Squid geen blokkeerlijstvermeldingen.

## Oorzaak

StevenBlack is een `hosts`-bestand: vermeldingen in het formaat `0.0.0.0 ads.example.com`. De Remote ACL-functie van OPNsense verwacht een tar.gz-archief met domeinlijsten in Squid ACL-formaat (één domein per regel). De formaten zijn incompatibel — OPNsense kan een hostsbestand niet parsen als een Squid ACL-lijst.

## Oplossing / workaround

Gebruik UT1 Toulouse (`dsi.ut-capitole.fr/blacklists`) in plaats daarvan. UT1 is de de-facto standaard voor op Squid gebaseerde categoriefiitering en wordt native ondersteund door OPNsense. Het distribueert tar.gz-archieven in Squid ACL-formaat met categorieën (gokken, malware, phishing, adult) die rechtstreeks aansluiten op de ACL-interface van OPNsense.

## Lessen

- OPNsense Remote ACL vereist tar.gz-archieven met domeinlijsten in Squid ACL-formaat
- Hostsbestandformaat (`0.0.0.0 hostnaam`) is niet compatibel
- UT1 Toulouse is de productiegeteste keuze voor deze gebruikssituatie

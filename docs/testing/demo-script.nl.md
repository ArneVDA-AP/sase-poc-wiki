---
title: "Demo-script (per rubric-criterium)"
tags: [testing, sase, demo, swg, ztna, casb, sd-wan]
---

# Demo-script (per rubric-criterium)

Een op zichzelf staand, interactief presentatiescript voor de eindpresentatie. Per
rubric-criterium staat er **wat je test**, **waarom die test dat punt bewijst**, **wat je
screencapt** en **wat je zegt** (voiceover). Drie pijlers krijgen een live demo van vijf minuten;
SD-WAN en de algemene beoordeling lopen via documentatie en de presentatie.

Deze pagina vult de technische testpagina's aan: het is de demo-/presentatielaag, niet de
testresultaten zelf:

- [Acceptatietests (F1–F15)](acceptance-tests.nl.md): de technische pass/fail-tests met commando's en verwachte output.
- [Aanvals- & bypass-scenario's](attack-scenarios.nl.md): de omzeilpogingen per pijler.
- **Deze pagina:** de rubric-gemapte demo-choreografie: presentator, timing, voiceover, screencap-instructies, de toestelmatrix en de enforce-validatie-checklist.

## Open het document

[**Open het interactieve demo-script ↗**](../demos/demo-script-rubric.html){target=_blank}

Het document is een zelfstandige HTML-pagina met een eigen vormgeving en interactieve elementen
(timers van vijf minuten per pijler, copy-to-clipboard-commandoblokken, klikbare checklists). Het
opent full-width in een nieuw tabblad zodat de layout en interactiviteit behouden blijven.

## Wat zit erin

- **Waar draai je wat:** een toestelmatrix die elke test koppelt aan de client waarop hij moet draaien (MOB-1 / SITE01 / enkel node), gegeven de huidige sandbox-opstelling.
- **Teststrategie & geconsolideerde resultaten:** de aanpak en de stand per pijler.
- **Per pijler** (Secure Web Gateway, CASB, ZTNA, SD-WAN, Algemene beoordeling): één criterium-kaart elk, met Test / Bewijst / Screencap / Voiceover en de terminalcommando's.
- **Enforce-validatie:** de laatste ronde die de punten "beslissing bewezen, afdwinging is de laatste stap" groen maakt (de eenmalige toggles per test).
- **Voorbereiding & documentatie:** het niet-demo-werk dat nodig is om de enforce-tests mogelijk te maken.

!!! note "Taal"
    Het demo-script zelf is in het Nederlands (het is het presentatiehulpmiddel van het team).
    Alleen deze wrapper-pagina is tweetalig; het gelinkte document wordt gedeeld over beide
    taalversies van de wiki.

## Gerelateerd

- [Testen: Acceptatietests](acceptance-tests.nl.md)
- [Testen: Aanvals- & bypass-scenario's](attack-scenarios.nl.md)
- [Concept: SASE](../concepts/sase.nl.md)

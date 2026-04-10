---
title: "Bevinding: ioc2rpz GUI JavaScript-aanmeldbug"
tags: [finding, ioc2rpz, workaround]
---

# Bevinding: ioc2rpz GUI JavaScript-aanmeldbug

**Component:** [ioc2rpz](../components/ioc2rpz.md)  
**Ernst:** Valkuil

## Wat er gebeurde

Inloggen op de ioc2rpz-GUI op `https://ioc2rpz.sandbox.local` mislukte stilzwijgend — de pagina leek het inlogverzoek even te verwerken, laadde vervolgens opnieuw zonder authenticatie. Er werd geen foutmelding getoond.

## Oorzaak

De `signIn`-functie in `/opt/ioc2rpz.gui/www/js/io2auth.js` mist een `e.preventDefault()`-aanroep. Wanneer de gebruiker het inlogformulier indient:
1. De browser dient het formulier native in (HTTP POST, veroorzaakt herlaad van pagina)
2. axios post ook de inloggegevens asynchroon
3. De native inzending herlaadt de pagina vóór voltooiing van axios, waardoor het authenticatieantwoord wordt afgebroken

Het resultaat: het axios-inlogverzoek wordt verzonden maar het antwoord wordt nooit verwerkt. De gebruiker ziet een pagina-herlaad zonder gevestigde sessie.

Dit is een upstream-bug in het ioc2rpz.gui-containerimage.

## Oplossing / workaround

Pas de correctie eenmalig toe na containerstart:

```bash
docker exec ioc2rpz-ioc2rpz-gui-1 sed -i \
  "s|signIn: function(e){ //|signIn: function(e){ e.preventDefault(); //|" \
  /opt/ioc2rpz.gui/www/js/io2auth.js
```

Dit moet opnieuw worden toegepast na elke containeropbouw (het image bevat de correctie niet).

## Lessen

- JavaScript-formulierinzending met zowel native inzending als axios vereist `e.preventDefault()` op de formulierverwerker — een veelvoorkomende frontend-bug
- Pas de correctie direct na containerstart toe, vóór eventuele inlogpogingen
- Documenteer de correctieopdracht in het operationele runbook — het is een terugkerende onderhoudsstap na containerrecreatie

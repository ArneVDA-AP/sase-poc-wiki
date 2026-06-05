---
title: "Bevinding: NATS JetStream maakt dubbel geneste jetstream/-directory"
tags: [finding, nats, jetstream, configuration]
---

# Bevinding: NATS JetStream maakt dubbel geneste jetstream/-directory

**Component:** [NATS JetStream](../components/nats-jetstream.md)  
**Ernst:** Valkuil

## Wat er gebeurde

Addendum J specificeerde `store_dir: /data/jetstream`. Na deployment maakte NATS geneste `jetstream/jetstream/`-directories aan onder `/data/`. Stream-introspectietools konden de data niet vinden op het verwachte pad, en handmatige inspectie onthulde de dubbele nesting.

## Oorzaak

NATS voegt automatisch `jetstream/` toe aan de geconfigureerde `store_dir`-waarde. Wanneer de configuratie al `jetstream/` bevat in het pad, is het resultaat dubbele nesting: `/data/jetstream/jetstream/`. Het addendum had redundant het achtervoegsel opgenomen dat NATS zelf toevoegt.

## Oplossing / workaround

Stel de correcte waarde in:

```
store_dir: /data
```

NATS maakt dan `/data/jetstream/` aan zoals verwacht. Dit werd empirisch bevestigd via het NATS-opstartlog (V32).

## Lessen

- Verifieer altijd NATS JetStream-opslagpaden in opstartlogs na deployment; de documentatie kan misleidend zijn over automatische pad-suffixing
- De dubbele nesting veroorzaakt geen dataverlies maar breekt aannames van tooling over waar streamdata zich bevindt
- Geef bij het configureren van JetStream alleen de bovenliggende directory op; NATS voegt zelf de `jetstream/`-subdirectory toe

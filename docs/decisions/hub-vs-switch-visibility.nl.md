---
title: "Beslissing: GNS3 Hub vs Switch voor volledige verkeerszichtbaarheid"
tags: [decision, zeek, network, gns3, architecture]
---

# Beslissing: GNS3 Hub vs Switch voor volledige verkeerszichtbaarheid

**Status:** Geïmplementeerd (PoC-gevalideerd op de parallelle stack, sandbox-integratie pending)  
**Datum:** April 2026 (Zeek/RITA implementatierapport)

> **Scope.** Deze beslissing hoort bij het Zeek/RITA-experiment op de **parallelle stack**
> (`mgmt01` = `192.168.122.20`). Het is nog niet in het sandbox-geheel geïntegreerd.

## Context

Zeek moet elk frame op een segment zien om behavioral analysis te doen. Een monitor-NIC op
`mgmt01` levert alleen bruikbare output op als elk frame op het core-segment die NIC ook echt
bereikt. In GNS3 was het core-segment oorspronkelijk een Ethernet Switch-node, en die switch
bleek het verkeerde bouwblok voor een monitoring-positie.

De ingebouwde Ethernet Switch van GNS3 is `ubridge`, een userspace learning L2-bridge. Zodra hij
MAC-adressen heeft geleerd forwardt hij unicast-frames alleen naar de bestemmingspoort, precies
zoals een productieswitch. Een monitor-poort op die switch ziet daardoor wel broadcast en zijn
eigen verkeer, maar niet de unicast-conversaties tussen andere nodes. De doorslaggevende
beperking: `ubridge` heeft geen CLI, geen management plane en geen SPAN- of port-mirror-functie.
Die mogelijkheid bestaat niet in de codebase, dus er is geen manier om de switch verkeer naar de
monitor-poort te laten kopiëren.

## Overwogen opties

| Optie | Voor | Tegen |
|-------|------|-------|
| **GNS3 Ethernet Switch (ubridge)** | Realistisch L2-gedrag; de standaardnode | Geen SPAN / port mirror; monitor-poort ziet geen inter-node unicast. Zeek is blind voor alles behalve broadcast. |
| **GNS3 Hub** | Floodt elk frame naar elke poort by design, dus de monitor-poort ontvangt al het verkeer; nul config | Niet hoe een productienetwerk werkt; introduceert flooding-overhead (verwaarloosbaar bij lab-verkeer van kbit/s tot lage Mbit/s) |
| **Open vSwitch (OVS) VM** | Echte L2-switching plus native SPAN via `ovs-vsctl` | Extra VM om te deployen en te onderhouden; overkill op deze schaal |

## Beslissing

Vervang de core Ethernet Switch door een GNS3 **Hub** (`Hub1`).

Een Hub floodt elk frame naar elke poort, wat betekent dat de monitor-NIC op `mgmt01` (`ens4`,
verbonden met `Hub1`-poort `Ethernet5`) een kopie ontvangt van elke conversatie op het
core-segment. Dit is het functionele equivalent van een SPAN- of mirror-poort op een managed
switch: al het verkeer is zichtbaar voor de monitoring-poort zonder enige switchconfiguratie,
want er valt niets te configureren. Bij het verkeersvolume van het lab is de flooding-overhead
niet meetbaar.

OVS is gedocumenteerd als de productiewaardige verfijning (echte switching met native SPAN), maar
is niet geïmplementeerd omdat de Hub hetzelfde monitoring-resultaat haalt op deze schaal met veel
minder bewegende delen.

## Gevolgen

- Het core-segment is geen getrouwe L2-switch meer. Voor een behavioral-analysis PoC is dat juist
  de bedoeling: volledige zichtbaarheid weegt zwaarder dan realistisch forwarden.
- De monitor-NIC moet in promiscuous mode draaien met NIC-offloading uitgeschakeld, anders geeft
  de kernel Zeek herassembleerde jumbo frames in plaats van wire frames. Zie de
  [Zeek & RITA-runbook](../runbooks/14-zeek-rita.nl.md) voor de `ens4`-setup.
- Dit lost zichtbaarheid op voor het **core**-segment alleen. Remote segmenten zitten achter hun
  eigen GNS3-switches en hebben een ander mechanisme nodig (kernel mirror in een GRE-tunnel). Het
  DC-segment loopt daar tegen een harde limiet aan, gedocumenteerd in
  [Bevinding: DC inner-segment mirror-limiet](../findings/dc-segment-mirror-limit.nl.md).
- Een productie-deployment zou de Hub vervangen door een OVS of ERSPAN-capable managed switch; de
  Zeek-kant verandert niet, alleen het frame-aanlevermechanisme.

## Gerelateerd

- [Component: Zeek](../components/zeek.nl.md)
- [Component: GNS3](../components/gns3.nl.md)
- [Concept: Behavioral analysis](../concepts/behavioral-analysis.nl.md)
- [Runbook: Zeek & RITA](../runbooks/14-zeek-rita.nl.md)
- [Bevinding: DC inner-segment mirror-limiet](../findings/dc-segment-mirror-limit.nl.md)

---
title: "Bevinding: FreeBSD kan niet kernel-mirroren, DC inner-segment zichtbaarheid is partieel"
tags: [finding, zeek, network, gre, opnsense]
---

# Bevinding: FreeBSD kan niet kernel-mirroren, DC inner-segment zichtbaarheid is partieel

**Component:** [Zeek](../components/zeek.nl.md)  
**Ernst:** Insight (geaccepteerde beperking voor de PoC)

> Waargenomen op de **parallelle stack** tijdens het Zeek/RITA-experiment (`mgmt01` =
> `192.168.122.20`, `OPNsense-1` = `192.168.122.11`, DC-segment `10.0.0.0/24`). Nog geen
> onderdeel van de main sandbox.

## Wat er gebeurde

Het Site1-segment werd in een GRE-tunnel gemirrord en bereikte Zeeks `worker-site1` netjes. Het
DC-segment werd op dezelfde manier opgezet: een GRE-tunnel van `OPNsense-1` naar `mgmt01`,
ontvangen als `gre-dc` en gevoed aan `worker-dc`. De tunnel zelf kwam op en Zeeks `tunnel.log`
toonde `Tunnel::GRE Tunnel::DISCOVER` ervoor, maar er arriveerden nooit gemirrorde frames op
`worker-dc`. De DC-worker zag de tunnel en niets erin.

## Oorzaak

`OPNsense-1` draait FreeBSD. FreeBSD heeft geen native kernel-level interface mirror die
equivalent is aan VyOS' `set interfaces ethernet ethX mirror ingress/egress`. Er is geen commando
om de kernel te vertellen "kopieer elk frame op `vtnet1` in deze GRE-tunnel."

Het voor de hand liggende Linux-alternatief vertaalt niet: `tcpdump`-output in een GRE-device
pipen faalt op FreeBSD omdat het GRE-tunneldevice correct GRE-geëncapsuleerde packets verwacht,
geen ruwe pcap-geformatteerde bytes. De tunnel bestaat dus wel, maar niets voedt het inner
DC-verkeer (`10.0.0.x` source/destination, vóór routing) erin.

## Oplossing / workaround

Accepteer partiële DC-zichtbaarheid voor de PoC. DC-verkeer wordt nog steeds op **core**-niveau
opgevangen door `worker-core` op `ens4` zodra het de WAN-interface van OPNsense op het
core-segment passeert. Wat verloren gaat is alleen het inner-segment-perspectief:
`10.0.0.x ↔ 10.0.0.x`-conversaties en het pre-routing source-IP van DC-hosts.

Dat verlies is hier acceptabel omdat het DC-segment exact twee nodes bevat: `OPNsense-1`
(`10.0.0.1`) en `Ubuntu-Server-DC-1` (`10.0.0.100`). Elke verbinding van DC-1 naar ergens buiten
de DC passeert OPNsense en wordt zichtbaar op het core-niveau. Er is geen tweede DC-host, dus er
is geen lateral movement om te missen. De totale topologie-dekking werd geschat op ~90%, met de
ontbrekende ~10% die precies dit DC inner-segment-verkeer is.

De productiefix is het frame-aanlevermechanisme vervangen, niet Zeek: een ERSPAN-capable managed
switch, of een Linux-gebaseerde gateway in plaats van FreeBSD/OPNsense, die beide native kernel
mirroring kunnen doen voor het DC-segment.

## Lessen

- Interface mirroring is OS-specifiek. VyOS (Linux) mirrort native; FreeBSD/OPNsense niet. Plan de
  capture-methode per gateway, niet één keer voor de hele topologie.
- GRE-tunnel discovery in Zeeks `tunnel.log` bevestigt de tunnel, niet het verkeer. Een
  `DISCOVER`-regel zonder payload op de worker betekent dat de andere kant niet daadwerkelijk
  mirrort.
- Een monitoring-gat is alleen zo erg als de topologie erachter. Met een single-server DC heeft de
  inner blind spot geen praktische beveiligingsimpact; in een multi-server DC zou die een
  ERSPAN-switch of een Linux-gateway nodig hebben.
- Dit is de tegenhanger van het werkende geval op het andere segment. Zie de VyOS-mirror in
  [Bevinding: VyOS GRE two-step commit](vyos-gre-two-step-commit.nl.md).

## Gerelateerd

- [Component: Zeek](../components/zeek.nl.md)
- [Beslissing: GNS3 Hub vs Switch](../decisions/hub-vs-switch-visibility.nl.md)
- [Bevinding: VyOS GRE two-step commit](vyos-gre-two-step-commit.nl.md)
- [Runbook: Zeek & RITA](../runbooks/14-zeek-rita.nl.md)

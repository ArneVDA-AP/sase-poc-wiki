---
title: "Bevinding: VyOS GRE-tunnel + mirror vereist twee aparte commits"
tags: [finding, vyos, gre, network, zeek]
---

# Bevinding: VyOS GRE-tunnel + mirror vereist twee aparte commits

**Component:** [VyOS](../components/vyos.nl.md)  
**Ernst:** Gotcha

> Geconfigureerd op de **parallelle stack** om het Site1-segment naar Zeek te mirroren (`VyOS-1` =
> `192.168.122.31` → `mgmt01` = `192.168.122.20`). Nog geen onderdeel van de main sandbox.

## Wat er gebeurde

Op `VyOS-1` bestaat de Site1-segmentmirror uit twee delen: een GRE-tunnel (`tun0`) gericht op
`mgmt01`, en een mirror-regel op `eth1` die ingress- en egress-frames in die tunnel kopieert.
Beide configureren en één enkele `commit` draaien faalt. VyOS weigert de commit omdat de
mirror-regel verwijst naar een tunnel-interface die op validatietijd nog niet bestaat.

## Oorzaak

VyOS valideert de mirror-doelinterface tijdens `commit`. Configuratiewijzigingen worden gestaged
en pas gerealiseerd als de commit slaagt, dus binnen één commit is de `tun0`-interface nog niet
aangemaakt op het moment dat de mirror-regel wordt gecheckt. De mirror-regel wijst naar een doel
dat VyOS niet kan vinden, en de hele commit wordt geweigerd. Het is een volgorde-beperking binnen
de commit, geen syntaxfout.

## Oplossing / workaround

Splits het in twee commits. Maak en commit eerst de tunnel, voeg dan de mirror-regels toe en
commit opnieuw:

```bash
configure

# Stap 1: maak de tunnel, commit dan zodat tun0 echt bestaat
set interfaces tunnel tun0 encapsulation 'gre'
set interfaces tunnel tun0 source-address '192.168.122.31'
set interfaces tunnel tun0 remote '192.168.122.20'
commit

# Stap 2: nu kunnen de mirror-regels naar tun0 verwijzen
set interfaces ethernet eth1 mirror ingress 'tun0'
set interfaces ethernet eth1 mirror egress 'tun0'
commit
save
```

Na stap 1 is de tunnel live, dus de mirror-regel in stap 2 valideert tegen een echte interface.

Twee parameternaam-vallen kwamen hier ook bovendrijven. VyOS wilde `source-address` en `remote`,
niet `local-ip` / `remote-ip`. De tunnel-syntax verschilt tussen VyOS-versies, dus check
`set interfaces tunnel tun0 ?` als een parameter geweigerd wordt. Het resultaat verifieert met:

```
tun0@NONE: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1476
    link/gre 192.168.122.31 peer 192.168.122.20
```

## Lessen

- VyOS commit-validatie is volgordegevoelig: een interface waarnaar verwezen wordt moet al bestaan
  (gecommit) voordat een regel ernaar kan wijzen. Als een regel van een interface afhangt, commit
  dan eerst de interface.
- "Maak de dependency, commit, maak dan de afhankelijke" is het algemene patroon voor elke
  VyOS-config waar de ene statement naar een andere resource verwijst.
- Tunnel-parameternamen zijn versie-afhankelijk. `set ... ?` wint van gokken.
- Dit is het segment dat *wel* werkte. Het DC-segment kon helemaal niet gemirrord worden omdat
  FreeBSD geen kernel-mirror heeft. Zie
  [Bevinding: DC inner-segment mirror-limiet](dc-segment-mirror-limit.nl.md).

## Gerelateerd

- [Component: VyOS](../components/vyos.nl.md)
- [Component: Zeek](../components/zeek.nl.md)
- [Bevinding: DC inner-segment mirror-limiet](dc-segment-mirror-limit.nl.md)
- [Runbook: Zeek & RITA](../runbooks/14-zeek-rita.nl.md)

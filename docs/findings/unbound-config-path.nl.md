---
title: "Bevinding: Unbound-configuratie moet in /usr/local/etc/unbound.opnsense.d/ staan"
tags: [finding, ioc2rpz, workaround]
---

# Bevinding: Unbound-configuratie moet in /usr/local/etc/unbound.opnsense.d/ staan

**Component:** [ioc2rpz](../components/ioc2rpz.md)  
**Ernst:** Blokker

## Wat er gebeurde

Een RPZ-configuratiebestand werd geplaatst op `/var/unbound/etc/rpz.conf`. Na een Unbound-herstart geactiveerd door OPNsense was de RPZ-configuratie verdwenen en startte Unbound zonder de RPZ-module.

## Oorzaak

`/var/unbound/` is de Unbound-chroot-map. OPNsense regenereert deze map bij elke Unbound-herstart — alle bestanden direct in `/var/unbound/etc/` geplaatst worden verwijderd en vervangen door de door OPNsense-template gegenereerde configuratie.

## Oplossing / workaround

Plaats persistente Unbound-configuratie in:
```
/usr/local/etc/unbound.opnsense.d/rpz.conf
```

OPNsense voegt alle `.conf`-bestanden uit deze map toe aan de gegenereerde Unbound-configuratie. Bestanden hier overleven herstarts.

De volledige RPZ-configuratie (`/usr/local/etc/unbound.opnsense.d/rpz.conf`):

```
server:
    module-config: "respip python validator iterator"
rpz:
    name: "threat-intel.rpz.sase"
    zonefile: "/var/unbound/threat-intel.rpz.sase.zone"
    primary: 127.0.0.1@53530
    allow-notify: 127.0.0.1
    rpz-action-override: nxdomain
    rpz-log: yes
    rpz-log-name: "ioc2rpz-threat-intel"
```

Opmerking: `zonefile` verwijst nog steeds naar `/var/unbound/` — dit is correct. Het zonebestandpad is waar Unbound de overgedragen zone opslaat, die het regenereert via zonetransfer. Alleen persistente *configuratie*bestanden moeten in `/usr/local/etc/unbound.opnsense.d/` staan.

## Lessen

- Plaats nooit persistente Unbound-configuratiebestanden in `/var/unbound/etc/` — OPNsense overschrijft deze map bij herstart
- Het juiste pad voor OPNsense Unbound-aanpassing is `/usr/local/etc/unbound.opnsense.d/*.conf`
- Dit is hetzelfde patroon als de `pre-auth/*.conf` van Squid — gebruik de aangewezen include-map, niet de chroot/template-map

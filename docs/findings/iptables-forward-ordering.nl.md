---
title: "Bevinding: iptables FORWARD-regelvolgorde met libvirt"
tags: [finding, network, workaround]
---

# Bevinding: iptables FORWARD-regelvolgorde met libvirt

**Component:** [GNS3](../components/gns3.md), [ioc2rpz](../components/ioc2rpz.md)  
**Ernst:** Valkuil

## Wat er gebeurde

Poortdoorsturing ACCEPT-regels werden toegevoegd aan de FORWARD-keten op de GNS3-host met `iptables -A FORWARD` (toevoegen). De poortdoorsturingen werkten niet — paketten werden gedropt ondanks dat overeenkomende regels aanwezig leken.

`iptables -L FORWARD --line-numbers` toonde de ACCEPT-regels op positie 24+, terwijl de `LIBVIRT_FWI`-keten op positie 22 stond met een REJECT-beleid.

## Oorzaak

libvirt installeert zijn eigen FORWARD-ketenregels die een REJECT bevatten voor verkeer dat niet door libvirt wordt beheerd. Wanneer regels worden toegevoegd (`-A FORWARD`), worden ze geplaatst na de libvirt REJECT-keten. De eerste overeenkomende regel wint — het REJECT van libvirt activeert vóór de ACCEPT-regel wordt bereikt.

## Oplossing / workaround

Altijd invoegen bovenaan de FORWARD-keten:

```bash
iptables -I FORWARD 1 -d 192.168.122.0/24 -j ACCEPT
```

`-I FORWARD 1` voegt in op positie 1 (vóór alle bestaande regels, inclusief de REJECT van libvirt).

Hetzelfde principe geldt voor alle ACCEPT-regels die bedoeld zijn om naast libvirt te werken:

```bash
# GUI-poortdoorsturingen voor pop01, mgmt01, site01
iptables -I FORWARD 1 -d 192.168.122.13 -j ACCEPT
iptables -I FORWARD 1 -d 192.168.122.23 -j ACCEPT
iptables -I FORWARD 1 -d 192.168.122.33 -j ACCEPT
```

## Lessen

- Gebruik bij het toevoegen van FORWARD-regels op een host met libvirt altijd `-I FORWARD 1` (invoegen bovenaan), nooit `-A FORWARD` (toevoegen)
- Diagnose: `iptables -L FORWARD --line-numbers` om regelposities en de libvirt-ketenlocatie te zien
- De REJECT-regel van libvirt is by design — het voorkomt dat onbeheerd verkeer libvirt-netwerken kruist

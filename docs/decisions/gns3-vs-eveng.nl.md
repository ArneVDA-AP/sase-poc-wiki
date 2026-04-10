---
title: "Beslissing: GNS3 vs EVE-NG"
tags: [decision, network, architecture]
---

# Beslissing: GNS3 vs EVE-NG

**Status:** Geïmplementeerd  
**Datum:** Begin van het project (vóór Verslag18)

## Context

Het handboek (v4) specificeerde EVE-NG als virtualisatieplatform. Het team moest meerdere VMs gelijktijdig draaien met vier mensen die tegelijk aan verschillende componenten werkten.

## Overwogen opties

| Optie | Voor | Tegen |
|-------|------|-------|
| **EVE-NG** | Handboek-gespecificeerd; webinterface (geen clientinstallatie) | Één webinterface — twee gebruikers die dezelfde node bewerken blokkeren elkaar; OVA-imageformaat incompatibel met QCOW2 (OPNsense/VyOS native formaat) |
| **GNS3** | Client-server-model — meerdere GUI-clients, dezelfde server; ondersteunt QCOW2/IMG/ISO native; multi-user gelijktijdig | Vereist installatie van desktop GUI-client op de laptop van elk teamlid |

## Beslissing

GNS3, draaiend als een systemd-service op een Ubuntu 24.04 VM (poc-1a, `10.158.10.67`) binnen Proxmox.

De multi-user-vereiste was de beslissende factor. Met vier teamleden die elk verantwoordelijk zijn voor een ander component, was sequentiële toegang (de beperking van EVE-NG) onpraktisch. Het client-server-model van GNS3 stelt alle vier leden in staat tegelijk verbinding te maken met dezelfde server op `10.158.10.67:3080` en onafhankelijk aan hun VMs te werken.

Het QCOW2-formaatcompatibiliteit was een secundaire factor — OPNsense, VyOS en Ubuntu 24.04 Server distribueren allemaal QCOW2 als primair formaat. De OVA-importvereiste van EVE-NG zou formaatconversie hebben vereist.

## Gevolgen

- GNS3 draait op Ubuntu 24.04 VM (geneste QEMU/KVM) — vereist Proxmox CPU `type = host` voor VMX/SVM-doorgifte
- libvirt-standaardnetwerk (`virbr0`, `192.168.122.0/24`) dient als WAN-segment en biedt NAT — de EVE-NG iptables NAT-procedures uit het handboek zijn niet van toepassing
- Handboek verwijst naar `192.168.100.0/24` als WAN-subnet; in deze sandbox is het `192.168.122.0/24` (libvirt standaard)
- GNS3 "Stop node" = QEMU SIGKILL — OPNsense moet worden afgesloten via `shutdown -h now` voordat GNS3 de node stopt (FreeBSD UFS soft-updates riskeert bestandssysteemcorruptie bij onregelmatige afsluiting)
- De EVE-NG-specifieke procedures uit het handboek (IP-forwarding, NAT-setup, OVA-imports) zijn allemaal vervangen door de GNS3-equivalenten gedocumenteerd in Addendum C

Zie ook: [Component: GNS3](../components/gns3.md)

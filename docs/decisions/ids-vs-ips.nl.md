---
title: "Beslissing: IDS-modus vs IPS-modus voor Suricata"
tags: [decision, suricata, network, opnsense]
---

# Beslissing: IDS-modus vs IPS-modus voor Suricata

**Status:** Geïmplementeerd (IDS-modus)  
**Datum:** April 2026 (Verslag23)

## Context

Suricata kan in twee modi werken:
- **IDS (Intrusion Detection System):** PCAP-opname — leest pakketkopieën passief, genereert alerts, kan geen verkeer droppen
- **IPS (Intrusion Prevention System):** Inline — bevindt zich in het verkeerspad, kan pakketten in realtime droppen

OPNsense biedt twee IPS-activeringsroutes: Netmap (native kernel bypass) en Divert (pf divert-to-regels).

## Overwogen opties

| Optie | Voor | Tegen |
|-------|------|-------|
| **IDS (PCAP-modus)** | Werkt op virtio-NIC's; geen NIC-driververeisten; stabiel | Kan verkeer niet in realtime droppen; bedreiging wordt gedetecteerd, niet geblokkeerd |
| **IPS via Netmap** | Echt pakketten droppen; laagste latentie inline-modus | Vereist NIC-drivers met native Netmap-ondersteuning (Intel igb/ixgbe, Broadcom bge). QEMU virtio-NIC's hebben geen Netmap-ondersteuning. Wanneer ingeschakeld: Suricata verwerkt 0 pakketten. |
| **IPS via Divert** | Geen speciale NIC-drivers vereist | Vereist expliciete `pf divert-to`-firewallregels; niet geconfigureerd in deze stack |

## Beslissing

IDS-modus (PCAP), met de drop/alert-beleidstabel geconfigureerd en klaar voor IPS-activering bij implementatie op fysieke hardware.

Netmap IPS werd getest en bevestigd niet-functioneel op virtio-NIC's. Wanneer Netmap IPS werd geactiveerd: Suricata rapporteerde 0 verwerkte pakketten en alle eerdere alertactiviteit stopte. Terugkeren naar PCAP IDS herstelde de normale werking. (Verslag23)

De gedifferentieerde beleidstabel (drop vs. alert per categorie) is geconfigureerd in OPNsense. Op fysieke hardware met ondersteunde NIC's vereist overschakelen naar IPS-modus alleen het inschakelen van de IPS-schakelaar — de policies zijn al correct.

## Gevolgen

- Suricata detecteert en waarschuwt, maar blokkeert geen verkeer in realtime in de huidige sandbox
- Drop-categorie-bedreigingen (emerging-malware, botcc, Abuse.ch) verschijnen in logs; blokkering vindt alleen plaats via Gate 3 (Unbound RPZ voor DNS, ClamAV voor downloads, Python DLP voor uploads)
- `procstat -f <PID> | grep bpf` is de juiste manier om te verifiëren dat Suricata opneemt — `sockstat` toont niets omdat Suricata BPF gebruikt, geen TCP/UDP-sockets
- Overschakelen naar IPS op fysieke hardware vereist `pf divert-to`-regels of native Netmap NIC-drivers

Zie ook: [Bevinding: Suricata Netmap/virtio](../findings/suricata-netmap-virtio.md), [Beslissing: Suricata WAN+LAN](suricata-wan-lan.md)

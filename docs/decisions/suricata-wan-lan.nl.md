---
title: "Beslissing: Suricata op WAN + LAN (niet wt0)"
tags: [decision, suricata, network, opnsense]
---

# Beslissing: Suricata op WAN + LAN (niet wt0)

**Status:** Geïmplementeerd  
**Datum:** Maart/april 2026 (Verslag22–23)

## Context

Suricata kan worden geconfigureerd om specifieke netwerkinterfaces te bewaken. De stack heeft drie interfaces op pop01: `vtnet0` (WAN), `vtnet1` (LAN/DC-LAN) en `wt0` (NetBird WireGuard-overlay). Welke interfaces moet Suricata bewaken?

## Overwogen opties

| Optie | Voor | Tegen |
|-------|------|-------|
| **vtnet0 (WAN) alleen** | Ziet al het internet-gebonden verkeer van pop01 | Mist intern DC-LAN-verkeer (laterale beweging, dc01-activiteit) |
| **vtnet0 + vtnet1 (WAN + LAN)** | Dekt zowel internet-gerichte als interne segmenten | Ziet geen gedecodeerd BYOD-verkeer (WireGuard-payload is niet zichtbaar) |
| **vtnet0 + vtnet1 + wt0** | Maximale dekking | wt0 is een WireGuard-tunnel; BPF op wt0 ziet 0 TCP-pakketten — WireGuard-verkeer verschijnt niet als inkomende frames vanuit pf's perspectief |

## Beslissing

Suricata op vtnet0 (WAN) en vtnet1 (LAN) alleen.

wt0 werd getest: `tcpdump -i wt0 -n 'tcp'` toonde 0 pakketten terwijl mobile01 actief was. WireGuard is een Laag 3 VPN — verkeer gerouteerd via wt0 verschijnt niet als inkomende frames op die interface vanuit BPF's perspectief. Suricata op wt0 zou niets zien.

vtnet1 (LAN) werd toegevoegd om bedreigingen op het interne DC-LAN-segment te detecteren — laterale beweging, afwijkend gedrag van dc01 zelf (bijv. software-updateverkeer). Dit dekt het "assume breach" zero-trust-principe: intern verkeer wordt niet standaard vertrouwd.

## Gevolgen

- Suricata vereist expliciete per-interface-declaraties in `custom.yaml` — het gebruik van `interface: default` neemt alleen vtnet0 op, niet vtnet1. Zie [Bevinding: Suricata interface default bug](../findings/suricata-interface-default-bug.md)
- BYOD-clientverkeer (gedecodeerd door Squid) is zichtbaar op vtnet0 als herencrypteerde TLS-sessies van Squid naar originele servers — Suricata extraheert TLS-metadata (SNI, JA3/JA4) hieruit
- WireGuard-verkeer van mobile01 verschijnt als versleuteld UDP/51820 op vtnet0 — de payload is niet inspecteertbaar door Suricata
- `HOME_NET` moet `100.64.0.0/10` (NetBird-overlay) bevatten zodat regels die `$HOME_NET` gebruiken NetBird-peers niet verkeerd classificeren als externe hosts

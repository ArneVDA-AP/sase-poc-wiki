---
title: "Beslissing: WPAD/PAC vs Transparante Proxy"
tags: [decision, proxy, wpad, squid, sase, network]
---

# Beslissing: WPAD/PAC vs Transparante Proxy

**Status:** Geïmplementeerd  
**Datum:** Maart 2026 (Verslag19)

## Context

BYOD-clients op de NetBird-overlay moeten al hun HTTP/HTTPS-verkeer via Squid laten lopen voor inspectie. Er zijn twee benaderingen: transparante interceptie (firewall leidt poort 80/443 stilzwijgend om) en expliciete proxy via WPAD/PAC (client is geconfigureerd om verkeer naar het proxyadres te sturen).

Het handboek specificeerde transparante proxy via `pf rdr`.

## Overwogen opties

| Optie | Voor | Tegen |
|-------|------|-------|
| **Transparante proxy (pf rdr)** | Geen clientconfiguratie vereist | Werkt niet op WireGuard-interface (`wt0`) — `pf rdr` ziet geen inkomend verkeer op Laag 3 VPN-interfaces |
| **WPAD/PAC expliciete proxy** | Werkt ongeacht het underlay-netwerk; correct SASE-patroon | Vereist dat client de proxyinstelling leest; niet-browserapps kunnen omzeilen |

## Beslissing

WPAD/PAC expliciete proxy, waarbij het PAC-bestand wordt geserveerd door Caddy op mgmt01 via `http://wpad.sandbox.local/wpad.dat`.

De transparante proxyoptie was geen levensvatbaar alternatief — het is technisch onmogelijk op de `wt0`-interface. WireGuard is een Laag 3 gerouteerde interface; `pf rdr` werkt op inkomende frames en ziet geen verkeer dat via `wt0` wordt doorgestuurd. Bevestigd via `tcpdump -i wt0 -n 'tcp'` die 0 pakketten toont (Verslag17; OPNsense GitHub issue #3857, gesloten als "not planned").

WPAD/PAC is ook het architecturaal correcte patroon voor een SASE-stack: het commerciële equivalent (Zscaler Client Connector) distribueert proxyconfiguratie naar de browser op precies dezelfde manier.

## Gevolgen

- mobile01 moet worden geconfigureerd met de PAC-bestand URL (handmatig via Windows-instellingen → Proxy → "Installatiescript gebruiken")
- Niet-browserapplicaties die de systeemproxyinstelling negeren, omzeilen Squid (gedeeltelijk gecompenseerd door Suricata op vtnet0)
- Squid moet luisteren op het NetBird overlay-IP (`100.70.154.79:3128`) via pre-auth include, niet via de GUI
- Caddy op mgmt01 serveert het PAC-bestand; de DNS-naam `wpad.sandbox.local` wordt opgelost via NetBird Custom DNS Zone

Zie ook: [Bevinding: wt0 pf rdr beperking](../findings/wt0-pf-rdr-limitation.md), [Bevinding: pre-auth ssl-bump parameters](../findings/pre-auth-ssl-bump-params.md)

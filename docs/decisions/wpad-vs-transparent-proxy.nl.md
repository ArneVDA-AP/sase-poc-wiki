---
title: "Beslissing: WPAD/PAC vs Transparante Proxy"
tags: [decision, proxy, wpad, squid, sase, network]
---

# Beslissing: WPAD/PAC vs Transparante Proxy

**Status:** Geïmplementeerd  
**Datum:** Maart 2026 (Verslag19)

## Context

Clients op de NetBird-overlay moeten al hun HTTP/HTTPS-verkeer via Squid laten lopen voor inspectie. Er zijn twee benaderingen: transparante interceptie (firewall leidt poort 80/443 stilzwijgend om) en expliciete proxy via WPAD/PAC (client is geconfigureerd om verkeer naar het proxyadres te sturen).

Het handboek specificeerde transparante proxy via `pf rdr`.

## Overwogen opties

| Optie | Voor | Tegen |
|-------|------|-------|
| **Transparante proxy (pf rdr)** | Geen clientconfiguratie vereist | Werkt niet op WireGuard-interface (`wt0`); `pf rdr` ziet geen inkomend verkeer op Laag 3 VPN-interfaces |
| **WPAD/PAC expliciete proxy** | Werkt ongeacht het underlay-netwerk; correct SASE-patroon | Vereist dat client de proxyinstelling leest; niet-browserapps kunnen omzeilen |

## Beslissing

WPAD/PAC expliciete proxy, waarbij het PAC-bestand wordt geserveerd door Caddy op mgmt01 via `http://wpad.sandbox.local/wpad.dat`.

Transparante interceptie via `pf rdr` is niet levensvatbaar *op de `wt0`-interface van pop01*. WireGuard is een Laag 3 gerouteerde interface; `pf rdr` werkt op inkomende frames en ziet geen verkeer dat via `wt0` wordt doorgestuurd. Bevestigd via `tcpdump -i wt0 -n 'tcp'` die 0 pakketten toont (Verslag17, Bevinding 17.10/17.11; OPNsense GitHub issue #3857, gesloten als "not planned"). Die beperking geldt specifiek voor redirecten op de firewall áchter de tunnel; ze sluit transparante interceptie elders in het pad niet uit.

WPAD/PAC is ook het architecturaal correcte patroon voor een SASE-stack: het commerciële equivalent (Zscaler Client Connector) distribueert proxyconfiguratie naar de browser op precies dezelfde manier.

WPAD/PAC blijft het **primaire** routingmechanisme. Het laat wel een gat: non-browser-apps die de systeemproxy negeren omzeilen Squid. Het vervolgonderzoek naar de transparante proxy dichtte dat gat anders, niet met `pf rdr` op `wt0`, maar met een **post-tunnel TPROXY op linuxpop01** (de exit node), die het gedecapsuleerde verkeer onderschept *na* de WireGuard-tunnel in plaats van op de firewall erachter. Dat omzeilt de `wt0`-beperking volledig en vangt al het verkeer ongeacht de clientconfiguratie. De diagnostiek achter die aanpak (het asymmetrische return-pad) is gevalideerd op de parallelle stack; het TPROXY-ontwerp zelf (Optie D) is ontworpen, maar nog niet in de sandbox geïntegreerd. WPAD/PAC en transparante proxy zijn dus **complementair**: WPAD/PAC configureert compliant verkeer, de transparante proxy is de enforcement-boundary voor al het verkeer. Zie [Component: Transparante proxy](../components/transparent-proxy.md).

## Gevolgen

- mobile01 moet worden geconfigureerd met de PAC-bestand URL (handmatig via Windows Settings → Proxy → "Use setup script")
- Niet-browserapplicaties die de systeemproxyinstelling negeren, omzeilen Squid (gedeeltelijk gecompenseerd door Suricata op vtnet0; volledig afgedekt door de complementaire [transparante proxy op linuxpop01](../components/transparent-proxy.md), die ontworpen is maar nog niet in de sandbox geïntegreerd)
- Squid moet luisteren op het NetBird overlay-IP (`100.70.154.79:3128`) via pre-auth include, niet via de GUI
- Caddy op mgmt01 serveert het PAC-bestand; de `sandbox.local` NetBird Custom DNS Zone routeert de query naar `wpad.sandbox.local` naar pop01 Unbound (de NetBird primary nameserver), die die resolvet naar de Caddy-host op mgmt01

Zie ook: [Component: Transparante proxy](../components/transparent-proxy.md), [Bevinding: wt0 pf rdr beperking](../findings/wt0-pf-rdr-limitation.md), [Bevinding: pre-auth ssl-bump parameters](../findings/pre-auth-ssl-bump-params.md)

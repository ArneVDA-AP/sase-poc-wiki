---
title: "Bevinding: wt0 pf rdr-beperking"
tags: [finding, network, proxy, squid]
---

# Bevinding: wt0 pf rdr-beperking

**Component:** [Squid](../components/squid.md)  
**Ernst:** Blokker

## Wat er gebeurde

Transparante proxy via `pf rdr` werd geconfigureerd om TCP-verkeer op de `wt0`-interface (NetBird WireGuard) te onderscheppen en door te sturen naar Squid. Verkeer van mobile01 via de NetBird-tunnel werd niet onderschept.

`tcpdump -i wt0 -n 'tcp'` toonde 0 paketten terwijl mobile01 actief aan het browsen was.

## Oorzaak

WireGuard creëert een Layer 3 gerouteerde interface. Verkeer dat via `wt0` wordt gerouteerd verschijnt vanuit het perspectief van de packet filter niet als inkomende frames op die interface. `pf rdr` werkt op inkomend verkeer — het kan via WireGuard doorgestuurd verkeer niet zien en dus ook niet omleiden.

Dit is een gedocumenteerde, bekende beperking. OPNsense GitHub issue #3857 — gesloten als "not planned".

## Oplossing / workaround

Overgeschakeld naar expliciete proxymodus met WPAD/PAC. Het PAC-bestand op `http://wpad.sandbox.local/wpad.dat` instrueert de browser om verkeer rechtstreeks naar `PROXY 100.70.154.79:3128` te sturen. Geen `pf rdr`-regel nodig.

Zie [Beslissing: WPAD/PAC vs. transparante proxy](../decisions/wpad-vs-transparent-proxy.md).

## Lessen

- WireGuard (Layer 3 VPN) is fundamenteel incompatibel met transparante `pf rdr`-proxy
- Voor elke op ZTNA-overlay gebaseerde architectuur is expliciete proxy met WPAD/PAC het correcte patroon
- `tcpdump` met TCP-filter op de vermoedelijke interface is de juiste diagnose: 0 paketten betekent dat de interface niet de plek is waar het verkeer verschijnt

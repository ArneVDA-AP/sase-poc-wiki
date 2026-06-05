---
title: "Bevinding: DC-LAN-isolatie via lege NetBird route-ACL-groep"
tags: [finding, netbird, vyos, network-route, isolation]
---

# Bevinding: DC-LAN-isolatie via lege NetBird route-ACL-groep

**Component:** [NetBird](../components/netbird.md), [VyOS](../components/vyos.md)  
**Ernst:** Inzicht

## Wat er gebeurde

DC-LAN-isolatie werd bereikt door een lege route-ACL-groep te gebruiken op een NetBird Network Route. Peers zonder expliciete groepsmatch krijgen geen toegang tot de route, ook al bestaat de route in de NetBird-configuratie. Het resultaat: poort 22 (SSH naar DC-LAN-hosts) is gesloten voor onbevoegde peers, terwijl internettoegang (0.0.0.0/0 exit node) open blijft.

## Oorzaak

NetBird Network Routes distribueren routeringsinformatie alleen naar peers die tot de opgegeven access control-groepen behoren. Een lege of niet-overeenkomende groepslijst betekent dat de route onzichtbaar is voor die peers: hun NetBird-client ontvangt de route nooit en probeert deze dus nooit te gebruiken. Dit is by-design gedrag, geen bug of misconfiguratie.

## Oplossing / workaround

Geen fix nodig; dit is intentioneel en biedt een schoon isolatiepatroon. Om een peer toegang te geven tot de DC-LAN-route, voeg deze toe aan de juiste access control-groep. Om toegang in te trekken, verwijder deze uit de groep.

Deze aanpak biedt selectieve netwerksegmentisolatie zonder aparte firewallregels op VyOS of de DC-LAN-hosts zelf.

## Lessen

- NetBird Network Routes fungeren tegelijkertijd als access control voor specifieke netwerksegmenten; benut lege route-ACL-groepen voor isolatiepatronen
- Dit is eleganter dan firewall-gebaseerde isolatie omdat het op de routeringslaag werkt: peers zonder toegang zien de route nooit, wat het aanvalsoppervlak verkleint
- Het patroon werkt voor elk netwerksegment achter een NetBird routing peer, niet alleen DC-LAN

---
title: "Bevinding: NetBird overlay-IPs zijn niet stabiel bij herinschrijving"
tags: [finding, netbird, identity-bridge, overlay]
---

# Bevinding: NetBird overlay-IPs zijn niet stabiel bij herinschrijving

**Component:** [Identity Bridge](../components/identity-bridge.md), [NetBird](../components/netbird.md)  
**Ernst:** Valkuil

## Wat er gebeurde

NetBird overlay-IPs verschuiven bij herinschrijvingscycli. Addendum H had IP `.95.98` hardcoded; na herinschrijving (peer verwijderen + opnieuw registreren) was het werkelijke IP `.218.100`. Na een daaropvolgende hernoeming en reboot verschoof het opnieuw naar `.64.80`. Elke configuratie of documentatie die naar het oude IP verwees brak stilzwijgend.

## Oorzaak

NetBird wijst overlay-IPs dynamisch toe vanuit zijn CGNAT-pool. Herinschrijving (een peer verwijderen en opnieuw registreren) wijst een nieuw IP toe uit de pool. Er is geen IP-reserveringsmechanisme in NetBird CE — de peer-identiteit is het peer-ID, niet het overlay-IP.

## Oplossing / workaround

De cache-invalidatielogica van de Identity Bridge moet op peer-ID gebaseerd zijn, niet op IP. Het `/api/peers`-endpoint retourneert zowel het stabiele peer-ID als het huidige overlay-IP, zodat de bridge identiteit kan volgen op peer-ID en zijn IP-mappings kan bijwerken wanneer ze veranderen.

Verwijs in configuratiebestanden en documentatie waar mogelijk naar peers op naam of peer-ID in plaats van overlay-IP.

## Lessen

- Hardcode nooit overlay-IPs in configuratie of documentatie — behandel ze als efemeer, vergelijkbaar met DHCP-leases
- Peer-ID is de enige stabiele identifier voor een NetBird-peer over herinschrijvingscycli heen
- Elk systeem dat overlay-IPs cachet of ernaar verwijst moet IP-wijzigingen graceful afhandelen

---
title: "Bevinding: Suricata Netmap IPS mislukt op virtio-NIC's"
tags: [finding, suricata, workaround]
---

# Bevinding: Suricata Netmap IPS mislukt op virtio-NIC's

**Component:** [Suricata](../components/suricata.md)  
**Ernst:** Blokker (voor IPS-modus)

## Wat er gebeurde

IPS-modus werd ingeschakeld in OPNsense (Intrusion Detection → Instellingen → IPS-modus ✔, patroonherkenner: Netmap). Suricata herstartte maar verwerkte 0 paketten. Alle eerder actieve alertcategorieën stopten met activeren. Het systeem leek te werken (geen crash) maar was volledig blind voor al het verkeer.

Terugkeren naar IDS-modus (PCAP) herstelde de normale alertactiviteit.

## Oorzaak

Netmap IPS-modus vereist NIC-stuurprogramma's met native Netmap-ondersteuning: Intel igb, ixgbe; Broadcom bge. Het QEMU virtio NIC-stuurprogramma (`vtnet`) heeft geen Netmap-ondersteuning.

Wanneer Netmap wordt ingeschakeld op een niet-ondersteunde NIC, start Suricata maar worden de Netmap-ringbuffers nooit gevuld. Suricata wacht op paketten die nooit aankomen. Er wordt geen fout gelogd — vanuit het perspectief van Suricata bestaat de interface en is Netmap geïnitialiseerd, maar de ring is simpelweg leeg.

`sysctl hw.model` op QEMU-VM's toont een generieke processorstring en helpt niet bij het identificeren van NIC-ondersteuning. De betrouwbare controle is `dmesg | grep -i features` voor SSE4.2 (nodig voor Hyperscan, niet voor Netmap), en het verifiëren van het NIC-stuurprogramma via `dmesg | grep vtnet`.

## Oplossing / workaround

Blijf in IDS-modus (PCAP). De gedifferentieerde drop/alert-beleidstabel is geconfigureerd en gereed. Op fysieke hardware met Intel igb/ixgbe of Broadcom bge NIC's kan IPS-modus worden geactiveerd door de schakelaar in te schakelen — het beleid is al correct.

## Lessen

- Netmap-ondersteuning is NIC-stuurprogrammaspecifiek, geen kernelmogelijkheid
- virtio-NIC's (de QEMU-standaard) ondersteunen Netmap niet — dit is een harde beperking
- Wanneer IPS-modus 0 paketten produceert zonder fout, is de eerste verdachte Netmap/NIC-incompatibiliteit
- Test altijd IPS-modusactivering in een lab vóór productie: terugkeren naar IDS-modus herstelt de vorige toestand onmiddellijk

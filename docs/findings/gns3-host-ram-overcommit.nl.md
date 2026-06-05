---
title: "Bevinding: RAM-overcommit op de GNS3-host met nul swap"
tags: [finding, gns3, qemu]
---

# Bevinding: RAM-overcommit op de GNS3-host met nul swap

**Component:** [GNS3](../components/gns3.md)  
**Ernst:** Blokkerend (structureel / latent)

## Wat gebeurde er

Tijdens een grote image pull op de mgmt01-guest meldde de GNS3-host (`poc-1a`) zichzelf tegen het geheugenplafond: 49,01 GB totaal, 46,33 GB gebruikt, 2,68 GB vrij, swap 0. De pull duwde de host-side resident set van mgmt01 naar ongeveer 12,3 GB (een download van ~2,4 GB plus uitpakken naar page cache), waardoor er nauwelijks ruimte overbleef op de hypervisor.

## Oorzaak

De som van de guest-RAM-toewijzingen op `poc-1a` overschrijdt het fysieke geheugen ruim. De sandbox-mgmt01 en de teamproject-mgmt01 vragen elk `-m 16384M`, pop01 neemt `8192M` en de overige nodes komen daar nog bovenop: samen ver voorbij de 49 GB die de host heeft. QEMU wijst guest-RAM lazy toe, dus deze overcommit blijft onzichtbaar zolang de guests hun pages niet aanraken. Een geheugenpiek die die pages alsnog afdwingt, zoals een image pull die inhoud naar de cache trekt, doet de marge instorten.

Met swap op 0 is er geen overloopruimte. Zodra het vrije geheugen op is, grijpt de Linux-OOM-killer in en beëindigt een willekeurig QEMU-proces. Dat slachtoffer kan elke guest op de host zijn: pop01 (een OPNsense-kill riskeert UFS-corruptie, zie de OPNsense stop-node-valkuil op de [GNS3](../components/gns3.md)-pagina), de sandbox-mgmt01 of de node van een teamlid op de gedeelde host. Eerdere metingen ("go-at-default, ruim vrij") werden binnen de guest gedaan en maten de verkeerde laag; de bindende constraint zit op de hypervisor, en de pre-flight had die gemist.

De grootste enkele scheefstand is de `-m 16384M` van mgmt01 tegenover ongeveer 1,1 GB werkelijk in gebruik: een reservering van 16 GB voor een guest die er een fractie van nodig heeft.

## Oplossing / workaround

Er is geen structurele fix toegepast. Voor de sessie is ruimte vrijgemaakt door sandbox-nodes te stoppen die op dat moment niet nodig waren (`sitepc01`, `dc01`), wat de host terugbracht onder het plafond. Het risico keert terug zodra guests weer parallel draaien, met name een toekomstige Zeek/RITA-node naast Wazuh.

Een duurzame fix heeft twee delen, beide vereisen overleg omdat ze de gedeelde host raken:

- Voeg een swapfile toe op `poc-1a`, zodat een OOM-situatie een overloopruimte heeft in plaats van een guest direct te killen. Dit raakt elk project op de host, dus stem het eerst af met het team.
- Verlaag de reservering van mgmt01 van 16384M naar ongeveer 8 GB, wat het grootste gat tussen gereserveerd en gebruikt dicht. Dit vereist een aparte guest-reboot.

## Lessen

- Meet `free -h` op de GNS3-host (`poc-1a`), niet alleen binnen de guest. Een guest die vrij geheugen meldt, zegt niets over de hypervisor-ruimte wanneer de toewijzingen overcommit zijn.
- De lazy RAM-toewijzing van QEMU verbergt overcommit tot een piek de pages alsnog afdwingt; een enkele grote image pull is genoeg om het bloot te leggen.
- Nul swap maakt van een geheugenpiek een OOM-kill van een willekeurig QEMU-proces. Op een gedeelde host kan dat de guest van een ander project neerhalen, niet alleen die van jezelf.

---
title: "Bevinding: NetBird config.json wordt 0 bytes na onzuivere afsluiting"
tags: [finding, network, workaround]
---

# Bevinding: NetBird config.json wordt 0 bytes na onzuivere afsluiting

**Component:** [NetBird](../components/netbird.md)  
**Ernst:** Valkuil

## Wat er gebeurde

Nadat GNS3 het pop01-knooppunt stopte met een SIGKILL (in plaats van een correcte `shutdown -h now`) startte NetBird niet op bij de volgende opstart zonder nuttige foutmelding. Interface `wt0` bestond niet. Alle services afhankelijk van het overlay-IP (`100.70.154.79`) — de Squid pre-auth listener, WPAD — waren niet functioneel.

`ls -la /var/db/netbird/config.json` toonde 0 bytes.

## Oorzaak

FreeBSD's UFS-bestandssysteem met soft updates schrijft journaalupdates asynchroon. Wanneer QEMU wordt gedood (GNS3 "Stop node" = SIGKILL naar het QEMU-proces), gaan gegevens in de schrijfbuffer die nog niet naar schijf zijn geleegd verloren. Het inode van het configuratiebestand bestaat (de bestandsnaam is bewaard) maar de inhoud was niet vastgelegd op schijf vóór de kill.

Dit is een bekend gedrag van UFS soft updates bij abrupte beëindiging, geen NetBird-bug.

## Oplossing / workaround

**Preventie:** Maak een back-up van config.json aan het einde van elke sessie:
```bash
cp /var/db/netbird/config.json /var/db/netbird/config.json.bak
```

**Herstel:** Als config.json 0 bytes is, herstel vanuit back-up:
```bash
cp /var/db/netbird/config.json.bak /var/db/netbird/config.json
service netbird restart
```

**Extra beperking:** Voeg `fsck_y_enable="YES"` toe aan `/etc/rc.conf` op pop01. FreeBSD voert dan automatisch fsck uit bij de volgende opstart na een onzuivere afsluiting, wat herstelt wat mogelijk is. Dit herstelt geen inhoud verloren uit schrijfbuffers, maar repareert wel inconsistenties in het bestandssysteem.

**Voorkoming aan de bron:** Sluit OPNsense altijd af via `shutdown -h now` vóór het stoppen van het GNS3-knooppunt.

## Lessen

- GNS3 "Stop node" = QEMU SIGKILL — behandel het als een stroomonderbreking
- FreeBSD UFS soft updates zijn kwetsbaar voor schrijfbufferverlies bij abrupte beëindiging
- Maak een back-up van kritieke configuratiebestanden (NetBird config.json, andere applicatiestatus) aan het einde van de sessie
- Het symptoom (0-byte configuratiebestand, service start niet) is kenmerkend — controleer dit eerst wanneer NetBird niet reageert na het stoppen van een GNS3-knooppunt

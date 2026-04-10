---
title: "Runbook: IDS"
tags: [runbook, suricata, ids, hyperscan]
---

# Runbook: IDS

**Bron:** `raw/Doc3_Suricata_IDS.md`
**Node(s):** pop01 (OPNsense — vtnet0 WAN + vtnet1 LAN)
**Vereisten:** [Runbook 03: Proxy & WPAD](03-proxy-wpad.nl.md) afgerond (Squid operationeel)
**Status:** Operationeel (IDS-modus — IPS gereed voor fysieke hardware)

---

## Vereistenchecklist

- [ ] pop01 heeft **minimaal 8 GB RAM** (zie Stap 0)
- [ ] OPNsense 25.1 op pop01
- [ ] vtnet0 (WAN) en vtnet1 (LAN) interfaces geconfigureerd
- [ ] Squid en ClamAV draaien al

---

## Stap 0: RAM controleren — minimaal 8 GB

> **Valkuil: Dit moet vóór alles anders gebeuren.** Bij 4 GB (of zelfs 6 GB) crashen ClamAV en Suricata stilzwijgend via FreeBSD's OOM-killer — er worden geen logboekregels geschreven. ClamAV gebruikt ~1,2 GB, Suricata ~760 MB na Hyperscan-compilatie (piekt op ~4 GB tijdens compilatie), Squid ~400 MB. Samen overschrijden ze 6 GB.

Als pop01 minder dan 8 GB heeft: sluit pop01 netjes af (`shutdown -h now`), verander het RAM in GNS3-knooppuntinstellingen, herstart.

**Verificatie na opstarten:**

```bash
top
# ClamAV: ~1230 MB RSS, Suricata: ~761 MB RSS
# Vrij: ~3974 MB
```

---

## Stap 1: SSE4.2 CPU-ondersteuning controleren

Hyperscan vereist SSE4.2. Gebruik niet `sysctl hw.model` — dit toont "QEMU Virtual CPU" wat niet informatief is.

```bash
dmesg | grep -i features
# Zoek naar: SSE4.2 in de Features2-regel
```

SSE4.2 aanwezig → Hyperscan is beschikbaar. De Proxmox-host geeft CPU-instructies door vanwege de `type=host` CPU-instelling (geconfigureerd in Runbook 01).

---

## Stap 2: Suricata configureren via GUI

Services → Intrusion Detection → Administration → Settings:

```
Ingeschakeld:      aangevinkt
IPS-modus:         niet aangevinkt  (IDS — zie opmerking hieronder)
Promiscuous mode:  aangevinkt
Pattern matcher:   Hyperscan
Interfaces:        WAN, LAN
Thuisnetwerken:    10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 100.64.0.0/10
```

> **Valkuil: `100.64.0.0/10` MOET in HOME_NET staan.** NetBird-peer-IP's gebruiken het CGNAT-bereik (100.64.x.x). Zonder dit subnet beschouwt Suricata NetBird-clients als externe hosts en worden regels die `$HOME_NET` gebruiken niet correct geactiveerd.

> **Valkuil: Promiscuous mode is essentieel op vtnet1 (LAN).** Zonder dit ziet de NIC alleen frames die bestemd zijn voor het MAC-adres van pop01. Verkeer van dc01 bestemd voor andere bestemmingen zou onzichtbaar zijn.

**Waarom IDS en niet IPS:** Netmap IPS vereist NIC-stuurprogramma's met native Netmap-ondersteuning (Intel `igb`, `ixgbe`). De virtio-NIC's in QEMU/GNS3 hebben geen Netmap-stuurprogramma — alle worker-threads verwerken nul pakketten. Het drop/alert-beleid is geconfigureerd en gereed voor IPS-activering op fysieke hardware met compatibele NIC's. Zie [Beslissing: IDS vs IPS](../decisions/ids-vs-ips.nl.md).

---

## Stap 3: Regelsets downloaden en inschakelen

Via Administration → Download, selecteer en klik op "Download & Update Rules":

| Regelset | Inhoud |
|----------|--------|
| ET Open (alle emerging-*-categorieën) | ~79.620 regels |
| Abuse.ch URLhaus | Malware-distributiedomeinen/-IP's |
| Abuse.ch SSL Fingerprint Blacklist | JA3-fingerprints van bekende malware TLS |
| Abuse.ch SSL IP Blacklist | Bekende malware C2-server IP's |

> **Valkuil: Gedownloade regels worden NIET automatisch geactiveerd.** Ga naar Administration → Rules, selecteer alle regelbestanden en schakel ze expliciet **in**. Dit is een aparte stap.

---

## Stap 4: Gedifferentieerd beleid aanmaken

Via Administration → Policies:

**Beleid 1 — Drop (hoge betrouwbaarheid, altijd kwaadaardig):**
- Categorieën: emerging-malware, emerging-exploit, emerging-worm, emerging-botcc, emerging-botcc.portgrouped, compromised, drop, ciarmy, Threatview_CS-C2, alle Abuse.ch-feeds
- Actie: Drop, Nieuwe actie: Drop

**Beleid 2 — Alert (risico op valse positieven):**
- Categorieën: tor, emerging-info, emerging-policy, emerging-dns, emerging-web_client, emerging-web_server, emerging-misc, emerging-games, emerging-chat, emerging-p2p, emerging-attack_response
- Actie: Alert, Nieuwe actie: Alert

Opmerking: `emerging-attack_response` is ingesteld op Alert — dit maakt testvalidatie met SID 2100498 mogelijk.

---

## Stap 5: custom.yaml aanmaken voor expliciete per-interface PCAP

OPNsense hergenerert `/var/unbound/suricata.yaml` bij elke GUI Apply. Gebruik in plaats daarvan het persistente aangepaste configuratiepad:

```bash
vi /usr/local/opnsense/service/templates/OPNsense/IDS/custom.yaml
```

> **Valkuil: `interface: default` resolveert naar vtnet0 ALLEEN.** Dit was de oorzaak van een volledige probleemoplossingsessie — vtnet1 had een BPF-apparaat open maar Suricata stuurde er nooit pakketten naartoe. Resultaat: 0 gebeurtenissen van vtnet1 ondanks bevestigde BPF-opname. Elke interface heeft een expliciete declaratie nodig.
> Zie [Finding: Suricata interface default-fout](../findings/suricata-interface-default-bug.nl.md).

Gebruik deze exacte inhoud — de definitieve werkende versie na de vtnet1-fix:

```yaml
pcap:
  - interface: vtnet0
    checksum-checks: no
  - interface: vtnet1
    checksum-checks: no
outputs:
  - eve-log:
      enabled: yes
      filetype: regular
      filename: eve.json
      types:
        - alert:
            tagged-packets: yes
        - anomaly:
            enabled: yes
        - drop:
            alerts: yes
            flows: start
        - ssh
        - flow
        - dns
        - http
        - tls:
            extended: yes
```

Toepassen:

```bash
configctl template reload OPNsense/IDS
configctl ids restart
```

---

## Stap 6: Controleren of Suricata opneemt

> **Valkuil: `sockstat` werkt NIET voor Suricata.** Suricata gebruikt BPF voor ruwe pakketopname, geen TCP/UDP-sockets. `sockstat -4 | grep suricata` geeft lege uitvoer, zelfs als Suricata correct draait.

**Correcte verificatie:**

```bash
# Haal PID op:
ps aux | grep suricata

# Controleer BPF-opname:
procstat -f <PID> | grep bpf
# Moet bpf0, bpf1 tonen — één per geconfigureerde interface
```

> **Valkuil: Logbestand heeft datumstempel.** Het pad is `/var/log/suricata/suricata_JJJJMMDD.log`, niet `suricata.log`.

```bash
cat /var/log/suricata/suricata_$(date +%Y%m%d).log | head -30
```

**RAM-piek tijdens Hyperscan-compilatie:** CPU stijgt naar 143%+ gedurende 1-2 minuten bij opstarten. Dit is normaal — wacht tot de compilatie klaar is.

---

## Eindverificatie

Voer deze vier tests uit om vier verschillende detectiecategorieën te bewijzen:

| # | Test | Opdracht | Verwachte SID | Interface |
|---|------|---------|--------------|-----------|
| 1 | Aanvalsrespons | `curl.exe -x http://100.70.154.79:3128 http://testmyids.com/` vanuit mobile01 | 2100498 | vtnet0 (WAN) |
| 2 | DNS-anomalieën | Automatisch — DNS .biz TLD-queries vanuit pop01 | 2027863 | vtnet0 (WAN) |
| 3 | Verdachte User-Agent | `curl.exe -x http://100.70.154.79:3128 -A "BlackSun" http://example.com` | 2008983 | vtnet0 (WAN) |
| 4 | LAN-verkeer | `apt update` op dc01 | 2013504 | vtnet1 (LAN) |

Controleer of vtnet1-gebeurtenissen bestaan:

```bash
grep '"in_iface":"vtnet1"' /var/log/suricata/eve.json | wc -l
# Moet > 0 zijn
```

Controleer OPNsense WebUI → Services → Intrusion Detection → Alerts: zowel WAN- als LAN-meldingen moeten zichtbaar zijn.

**Opmerking over herhaalde tests:** Meerdere curl-verzoeken naar hetzelfde domein kunnen slechts één melding opleveren. Dit is geen drempelwaardefout — Squid hergebruikt upstream TCP-verbindingen (connection pooling), dus Suricata ziet één flow en genereert correct één melding per SID per flow.

---

## Checklist

- [ ] pop01 heeft 8 GB RAM, alle services stabiel
- [ ] SSE4.2 bevestigd, Hyperscan geselecteerd
- [ ] `100.64.0.0/10` in HOME_NET
- [ ] 79.620+ regels gedownload EN ingeschakeld
- [ ] Drop/Alert-beleid geconfigureerd
- [ ] `procstat -f <PID> | grep bpf` toont bpf0 en bpf1
- [ ] vtnet0-meldingen aanwezig (SID 2100498 of vergelijkbaar)
- [ ] vtnet1-meldingen aanwezig (dc01-verkeer)

---

## Gerelateerd

- [Component: Suricata](../components/suricata.nl.md)
- [Beslissing: IDS vs IPS](../decisions/ids-vs-ips.nl.md)
- [Beslissing: Suricata WAN+LAN](../decisions/suricata-wan-lan.nl.md)
- [Finding: Suricata interface default-fout](../findings/suricata-interface-default-bug.nl.md)
- [Finding: Suricata Netmap/virtio](../findings/suricata-netmap-virtio.nl.md)

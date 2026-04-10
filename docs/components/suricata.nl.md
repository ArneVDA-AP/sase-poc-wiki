---
title: "Suricata IDS — WAN + LAN-netwerkinspectie"
tags: [suricata, opnsense, ids, ips, network, firewall, sase]
---

# Suricata IDS — WAN + LAN-netwerkinspectie

**Rol:** Parallelle netwerklaagsinspsectie op pop01 — detecteert bedreigingen in ruwe pakketstromen op WAN (vtnet0) en LAN (vtnet1) met behulp van handtekeninggebaseerde regels. Complementair aan de ICAP-pijplijn, die HTTP-inhoud inspecteert; Suricata inspecteert netwerkstromen die de proxy volledig omzeilen.  
**Versie:** Suricata 7.x (meegeleverd met OPNsense 25.1)  
**Configuratielocatie:** `/usr/local/opnsense/service/templates/OPNsense/IDS/custom.yaml` (persistente configuratie), `/var/log/suricata/suricata_JJJJMMDD.log`, `/var/log/suricata/eve.json`

---

## Werking in deze stack

Suricata draait in PCAP-opnamemodus op pop01 en leest ruwe frames van BPF-apparaten op vtnet0 (WAN) en vtnet1 (LAN). Het stelt TCP-streams opnieuw samen, parseert protocolvelden en vergelijkt met 79.620+ regels uit vier regelsets (ET Open, Abuse.ch URLhaus, SSL Fingerprint Blacklist, SSL IP Blacklist).

**Wat vtnet0 (WAN) ziet:** De opnieuw versleutelde upstream HTTPS-verbindingen van Squid, DNS-query's van de resolver van pop01, overig direct TCP/UDP-verkeer dat de proxy omzeilt. Suricata extraheert TLS-metadata (SNI, JA3/JA4-vingerafdrukken), HTTP-headers in platte tekst (van niet-HTTPS-stromen), DNS-query/response-gegevens en vergelijkt met C2/botnet IP- en domeinhandtekeningen.

**Wat vtnet1 (LAN) ziet:** Al het interne DC-LAN-verkeer van dc01 — onversleuteld, alle protocollen. Dit detecteert bedreigingen op het interne segment (laterale beweging, verkenning, software-update-afwijkingen van dc01).

**Waarom niet wt0 (NetBird):** WireGuard is een Layer 3 VPN. Verkeer dat via wt0 wordt gerouteerd verschijnt vanuit het perspectief van BPF niet als inkomende frames op die interface. `tcpdump` op wt0 met een TCP-filter toont 0 paketten. Suricata op wt0 zou niets zien. Zie [Bevinding: wt0 pf rdr-beperking](../findings/wt0-pf-rdr-limitation.md) en [Beslissing: Suricata WAN+LAN](../decisions/suricata-wan-lan.md).

**IDS-modus, niet IPS:** Suricata draait in IDS-modus (PCAP, geen pakketdropping). Netmap IPS-modus vereist NIC-stuurprogramma's met native Netmap-ondersteuning (Intel igb/ixgbe, Broadcom bge) — virtio-NIC's in QEMU hebben geen van deze. Divert IPS vereist expliciete `pf divert-to`-firewallregels die niet zijn geconfigureerd. De drop/alert-beleidstabel is geconfigureerd en gereed voor IPS-activering op fysieke hardware. Zie [Beslissing: IDS vs. IPS](../decisions/ids-vs-ips.md) en [Bevinding: Suricata Netmap/virtio](../findings/suricata-netmap-virtio.md).

**RAM-vereiste:** Suricata + ClamAV + Squid gelijktijdig uitvoeren vereist **minimaal 8 GB RAM** op pop01. Hyperscan-patrooncompilatie van ET Open-regels piekt op ~4 GB RSS. Bij 6 GB beëindigt de OOM-killer services stilzwijgend zonder loguitvoer.

---

## Configuratie

### GUI-instellingen

Services → Intrusion Detection → Administratie → Instellingen:

```
Ingeschakeld:       ✔
IPS-modus:          ☐  (IDS)
Promiscue modus:    ✔
Patroonherkenner:   Hyperscan
Interfaces:         WAN, LAN
Thuisnetwerken:     10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 100.64.0.0/10
```

`100.64.0.0/10` in HOME_NET is cruciaal: NetBird-client-IP's vallen in het CGNAT-bereik. Zonder dit item behandelt Suricata NetBird-peers als externe hosts en worden regels met `$HOME_NET` onjuist geactiveerd.

**Hyperscan-selectie:** Controleer of SSE4.2 beschikbaar is voordat u Hyperscan selecteert. Gebruik `dmesg | grep -i features` en zoek naar `SSE4.2` in Features2. `sysctl hw.model` op QEMU-VM's toont een generieke string en is niet betrouwbaar.

### Regelsets

Downloaden via Administratie → Download. Na het downloaden zijn regels **niet automatisch actief** — ga naar Administratie → Regels en schakel alle regelbestanden afzonderlijk in.

| Regelset | Regels |
|----------|--------|
| ET Open (alle emerging-*-categorieën) | ~79.620 |
| Abuse.ch URLhaus | malware-distributie |
| Abuse.ch SSL Fingerprint Blacklist | JA3-vingerafdrukken |
| Abuse.ch SSL IP Blacklist | bekende C2-IP's |

### Gedifferentieerd drop/alert-beleid

Via Administratie → Beleid. Geconfigureerd en gereed voor IPS-activering:

| Actie | Categorieën |
|-------|------------|
| **Drop** | emerging-malware, emerging-exploit, emerging-worm, emerging-botcc, emerging-botcc.portgrouped, compromised, drop (Spamhaus), ciarmy, Threatview Cobalt Strike C2, alle Abuse.ch-feeds |
| **Alert** | tor, emerging-info, emerging-policy, emerging-dns, emerging-web_client, emerging-web_server, emerging-misc, emerging-games, emerging-chat, emerging-p2p, emerging-attack_response |

Drop-categorieën zijn die waarbij een overeenkomst **altijd** kwaadaardig is — geen legitiem verkeer mogelijk. Alert-categorieën hebben risico op valse positieven.

### custom.yaml (persistente configuratie)

OPNsense regenereert `suricata.yaml` bij elke GUI-toepassing. Persistente wijzigingen gaan in:

```
/usr/local/opnsense/service/templates/OPNsense/IDS/custom.yaml
```

**Definitieve werkende inhoud (na vtnet1-correctie):**

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

## Validatieresultaten

Vier detectiecategorieën bevestigd na vtnet1-correctie:

| # | Categorie | Test | SID | Interface |
|---|-----------|------|-----|-----------|
| 1 | Aanvalsrespons | `curl.exe -x http://100.70.154.79:3128 http://testmyids.com/` | 2100498 | vtnet0 |
| 2 | DNS-afwijking | Automatisch — DNS .biz TLD-query's van pop01-resolver | 2027863 | vtnet0 |
| 3 | Verdachte user-agent | `curl.exe -x ... -A "BlackSun" http://example.com` | 2008983 | vtnet0 |
| 4 | Softwaredetectie (LAN) | dc01 `apt update`-activiteit | 2013504 | vtnet1 |

---

## Integratiepunten

| Component | Richting | Wat |
|-----------|----------|-----|
| [Squid](squid.md) | parallel | Suricata op vtnet0 ziet de upstreamverbindingen van Squid — opnieuw versleuteld HTTPS; TLS-metadata zichtbaar |
| [NetBird](netbird.md) | HOME_NET-afhankelijkheid | `100.64.0.0/10` moet in HOME_NET staan voor correcte regelevaluatie |
| [ioc2rpz/RPZ](ioc2rpz.md) | aanvullend | Abuse.ch URLhaus in Suricata-handtekeningen en RPZ-feeds overlappen in bron; Suricata detecteert verbindingen, RPZ voorkomt DNS-resolutie |

---

## Bekende problemen / valkuilen

**vtnet1 `interface: default` genereert geen gebeurtenissen** — het gebruik van `interface: default` in custom.yaml resulteert in alleen de primaire opname-interface (vtnet0), niet alle interfaces. vtnet1 had een BPF-apparaat open maar ontving geen paketten. Oplossing: expliciete per-interface-declaraties. Zie [Bevinding: Suricata interface default-bug](../findings/suricata-interface-default-bug.md).

**`sockstat` werkt niet voor Suricata-verificatie** — Suricata gebruikt BPF, geen TCP/UDP-sockets. `sockstat -4 | grep suricata` retourneert niets, ook als Suricata correct draait. Gebruik in plaats daarvan `procstat -f <PID> | grep bpf`.

**Logbestand heeft datumstempel** — `/var/log/suricata/suricata.log` bestaat niet. Het juiste pad is `/var/log/suricata/suricata_JJJJMMDD.log`.

**Squid-verbindingspooling verklaart enkele alerts per test** — wanneer dezelfde testmyids.com-URL meerdere keren via Squid wordt opgevraagd, ziet Suricata slechts één stroom (Squid hergebruikt de upstream TCP-verbinding). Dit genereert één alert per SID per stroom — correct gedrag, geen drempelprobleem.

**Netmap IPS mislukt op virtio-NIC's** — zie [Bevinding: Suricata Netmap/virtio](../findings/suricata-netmap-virtio.md). Overschakelen naar Netmap zorgt voor nul verwerkte paketten en maakt alle eerder actieve alertactiviteit ongedaan.

---

## Gerelateerd

- [Architectuuroverzicht](../overview/architecture.md)
- [Concept: Zero Trust](../concepts/zero-trust.md)
- [Beslissing: Suricata WAN+LAN](../decisions/suricata-wan-lan.md)
- [Beslissing: IDS vs. IPS](../decisions/ids-vs-ips.md)
- [Bevinding: Suricata interface default-bug](../findings/suricata-interface-default-bug.md)
- [Bevinding: Suricata Netmap/virtio](../findings/suricata-netmap-virtio.md)
- [Bevinding: wt0 pf rdr-beperking](../findings/wt0-pf-rdr-limitation.md)

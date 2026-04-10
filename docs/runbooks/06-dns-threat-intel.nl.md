---
title: "Runbook: DNS Threat Intelligence"
tags: [runbook, ioc2rpz, rpz, bind, unbound, dns]
---

# Runbook: DNS Threat Intelligence

**Bron:** `raw/Doc4_DNS_Threat_Intelligence.md`
**Node(s):** mgmt01 Docker (ioc2rpz + GUI) + pop01 (BIND 9.20 + Unbound RPZ)
**Vereisten:** [Runbook 02: ZTNA Overlay](02-ztna-overlay.nl.md) afgerond (NetBird DNS-relay operationeel)
**Status:** Operationeel — 71.767 RPZ-records, drie testpunten gevalideerd

---

## Vereistenchecklist

- [ ] NetBird overlay operationeel (Runbook 02)
- [ ] mgmt01 Docker-daemon draaiend
- [ ] pop01 Unbound lost DNS-queries op
- [ ] Netwerkpad mgmt01 → pop01 via 192.168.122.x

---

## Stap 1: ioc2rpz + GUI uitrollen op mgmt01

**Los eerst poortconflicten op.** Op mgmt01 wordt poort 53 geclaimd door `systemd-resolved` en de NetBird DNS-relay. Bind ioc2rpz aan het beheer-IP `192.168.122.23`.

Maak `/opt/ioc2rpz/docker-compose.yml` aan:

```yaml
version: '3'
services:
  ioc2rpz:
    image: pvmdel/ioc2rpz
    ports:
      - "192.168.122.23:53:53/udp"
      - "192.168.122.23:53:53/tcp"
      - "192.168.122.23:853:853/tcp"
      - "192.168.122.23:8443:8443/tcp"
    volumes:
      - ./cfg:/opt/ioc2rpz/cfg
      - ./db:/opt/ioc2rpz/db
    restart: always
    logging:
      driver: syslog
    depends_on:
      - ioc2rpz-gui

  ioc2rpz-gui:
    image: pvmdel/ioc2rpz.gui
    ports:
      - "192.168.122.23:8080:80/tcp"
      - "192.168.122.23:8444:443/tcp"
    volumes:
      - ./cfg:/opt/ioc2rpz.gui/export-cfg
      - ./db:/opt/ioc2rpz.gui/www/io2cfg
      - ./ssl:/etc/apache2/ssl
    restart: always
    logging:
      driver: syslog
```

Vernieuw het SSL-certificaat (het meegeleverde certificaat verliep in juli 2022):

```bash
sudo openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
  -keyout /opt/ioc2rpz/ssl/ioc2_server.key \
  -out /opt/ioc2rpz/ssl/ioc2_server.pem \
  -subj "/CN=ioc2rpz.sandbox.local"
```

```bash
cd /opt/ioc2rpz
docker compose up -d
```

---

## Stap 2: iptables FORWARD-regels configureren voor GUI-toegang

> **Valkuil: Gebruik altijd `-I FORWARD 1`, nooit `-A FORWARD`.** De FORWARD-keten op de GNS3-host heeft een libvirt `LIBVIRT_FWI`-keten op positie ~22 die nieuwe verbindingen WEIGERT voordat toegevoegde regels worden bereikt. Je ziet 0 pakketten op je ACCEPT-regels — de meest verborgen valkuil van de hele implementatie.
> Zie [Finding: iptables FORWARD-volgorde](../findings/iptables-forward-ordering.nl.md).

```bash
sudo iptables -I FORWARD 1 -d 192.168.122.23 -p tcp --dport 8080 -j ACCEPT
sudo iptables -I FORWARD 1 -d 192.168.122.23 -p tcp --dport 8444 -j ACCEPT
```

Bewaar met `netfilter-persistent`.

---

## Stap 3: ioc2rpz GUI JavaScript-fout herstellen

Inloggen mislukt met "Unknown error!!!" in de browser, terwijl `curl` naar `/io2auth.php/signin` `authSuccess` teruggeeft.

**Oorzaak:** De `signIn`-functie in `io2auth.js` mist `e.preventDefault()`. De browser doet een native formuliersubmit (synchrone paginaverversing) die de asynchrone axios POST onderbreekt.

```bash
sudo docker exec ioc2rpz-ioc2rpz-gui-1 sed -i \
  "s|signIn: function(e){ //|signIn: function(e){ e.preventDefault(); //|" \
  /opt/ioc2rpz.gui/www/js/io2auth.js
```

Hard vernieuwen (Ctrl+Shift+R) na de fix. Zie [Finding: ioc2rpz GUI JS-fout](../findings/ioc2rpz-gui-js-bug.nl.md).

> **Valkuil: Deze patch is niet persistent.** Hij leeft in de containerlaag. Een `docker compose down && up` met een nieuw image verliest de patch. Pas opnieuw toe na containerrebuilds.

**Wachtwoordvereisten:** >7 tekens + minstens 1 cijfer, 1 kleine letter, 1 hoofdletter, 1 speciaal teken. OF >15 tekens.

**Caddy reverse proxy** voor browsertoegang via `ioc2rpz.sandbox.local` — voeg toe aan het Caddyfile op mgmt01:

```caddyfile
ioc2rpz.sandbox.local {
    tls internal
    reverse_proxy https://192.168.122.23:8444 {
        transport http {
            tls_insecure_skip_verify
        }
    }
}
```

---

## Stap 4: ioc2rpz-server, TSIG, feeds en RPZ-zone configureren

Via de GUI na inloggen:

**Serverconfiguratie:**
```
Naam:       sase-poc-rpz
MGMT IP:    172.20.0.3 (Docker intern)
Publiek IP: 192.168.122.23
NS:         ns1.ioc2rpz.local
ACL:        127.0.0.1, 172.20.0.2 (GUI-container), 172.20.0.1
```

**TSIG-sleutels (maak er twee aan):**
```
tkey_mgmt_1        — beheersleutel, hmac-md5
tkey_rpz_transfer  — zonetransfersleutel, hmac-sha256
```

**Bronnen (Threat Feeds):**
```
urlhaus_rpz:   https://urlhaus.abuse.ch/downloads/rpz/
threatfox_rpz: https://threatfox.abuse.ch/downloads/threatfox.rpz
```

Opmerking: de ThreatFox RPZ-URL is `/downloads/threatfox.rpz`, NIET `/downloads/rpz/` (dat geeft 404).

**RPZ-zone:**
```
Naam:       threat-intel.rpz.sase
Actie:      nxdomain
Cache:      true
Wildcards:  true
TSIG-sleutel: tkey_rpz_transfer
Bronnen:    urlhaus_rpz + threatfox_rpz
Notify:     192.168.122.13  (pop01)
AXFR-tijd:  3600
```

> **Valkuil: TSIG-sleutel MOET gekoppeld zijn aan de zone.** Een niet-gekoppelde sleutel (leeg `tkeys`-veld in de configuratie) veroorzaakt zonetransferafwijzing met "tsig indicates error". Controleer in de GUI of de zone de juiste TSIG-sleutel toont, klik dan op Publish.

Na Publish, verwachte uitvoer:
```
Source: "urlhaus_rpz", got 450 indicators
Source: "threatfox_rpz", got 35475 indicators
Zone "threat-intel.rpz.sase" updated, serial ..., 71836 rules, 35918 indicators
```

---

## Stap 5: BIND 9.20 installeren op pop01

Via System → Firmware → Plugins: installeer `os-bind`. Na installatie: Services → BIND verschijnt in het menu.

> **Valkuil: Maak de ACL aan vóór Algemene Instellingen.** Het ACL-dropdown in Algemene Instellingen toont alleen ACL's die al bestaan. Als je eerst Algemene Instellingen configureert, is het ACL-dropdown leeg.

**ACL:** Services → BIND → Access Control List:
```
Naam:     loopback_only
Netwerken: 127.0.0.1/32
```

Navigeer weg en terug (GUI-dropdown quirk).

**Algemene instellingen:**
```
BIND Daemon inschakelen: aangevinkt
Luister-IP's:           127.0.0.1
Luisterpoort:           53530
DNS Forwarders:         leeg
Recursie:               geen ACL (uit)
DNSSec Validatie:       nee
Overdracht toestaan:    loopback_only
```

**Secundaire zone:** Services → BIND → Secondary Zones:
```
Zonenaam:           threat-intel.rpz.sase
Primair IP:         192.168.122.23
Primaire poort:     53
Transfersleutel:    tkey_rpz_transfer
Transfer Sleutel Algo: hmac-sha256
Transfer Sleutelwaarde: [base64 vanuit ioc2rpz GUI → Configuration → TSIG Keys]
Notify toestaan:    192.168.122.23
Query toestaan:     loopback_only
```

Zie [Beslissing: BIND als TSIG-intermediair](../decisions/bind-tsig-intermediary.nl.md) waarom BIND nodig is (Unbound 1.24.2 mist TSIG-ondersteuning).

**Controleer zonetransfer:**

```bash
pluginctl bind status
cat /var/log/named/named.log | tail -20
```

Verwacht:
```
Transfer completed: 97 messages, 71767 records, 1557644 bytes, 0.645 secs
```

**TSIG-probleemoplossing** — als `tsig indicates error` in named.log:
1. Controleer klokverschil: `date -u` op zowel pop01 als mgmt01 (max 300s tolerantie)
2. Verifieer dat sleutelnaam exact overeenkomt tussen ioc2rpz en BIND-configuraties
3. In ioc2rpz GUI: zorg dat de TSIG-sleutel gekoppeld is aan de zone (niet alleen aangemaakt)

---

## Stap 6: Unbound RPZ activeren op pop01

> **Valkuil: Configuratiepad is `/usr/local/etc/unbound.opnsense.d/rpz.conf`, NIET `/var/unbound/etc/rpz.conf`.** De `/var/unbound/` chroot wordt hergenererd bij elke Unbound-herstart — bestanden die daar geplaatst worden, worden verwijderd.
> Zie [Finding: Unbound configuratiepad](../findings/unbound-config-path.nl.md).

```bash
vi /usr/local/etc/unbound.opnsense.d/rpz.conf
```

Inhoud:

```
server:
    module-config: "respip python validator iterator"
rpz:
    name: "threat-intel.rpz.sase"
    zonefile: "/var/unbound/threat-intel.rpz.sase.zone"
    primary: 127.0.0.1@53530
    allow-notify: 127.0.0.1
    rpz-action-override: nxdomain
    rpz-log: yes
    rpz-log-name: "ioc2rpz-threat-intel"
```

> **Valkuil: `python` MOET in de module-config staan.** De ingebouwde OPNsense DNS-bloklijst draait als een Python-script in Unbound. Zonder `python` werkt de OPNsense DNSBL niet meer.

> **Valkuil: `configctl unbound check` geeft een vals positief.** `unbound-checkconf` heeft een vaste whitelist van modulecombinaties; `respip + python` staat er niet in maar werkt tijdens runtime. Negeer de fout en herstart direct:

```bash
configctl unbound restart
```

**Controleer het laden van modules:**

```bash
cat /var/log/system.log | grep "init module"
# Moet alle 4 modules tonen:
# init module 0: respip
# init module 1: python
# init module 2: validator
# init module 3: iterator
```

De OPNsense WebUI toont "The configuration contains manual overwrites" — dit is verwacht en correct.

---

## Stap 7: NetBird DNS primaire nameserver configureren

Deze stap is een **vereiste voor BYOD RPZ-bescherming**. Zonder dit omzeilen externe queries van mobile01 Unbound volledig.

NetBird Dashboard → DNS → Nameservers → Add:

```
IP:             100.70.154.79 (pop01 overlay-IP)
Match-domeinen: (LEEG LATEN)
```

> **Valkuil: Lege match-domeinen = primaire nameserver voor ALLE queries.** Niet alleen `*.sandbox.local`. Zonder dit gaan queries voor externe domeinen via de originele DNS van de adapter, en beschermt RPZ niet tegen externe kwaadaardige domeinen.
> Zie [Finding: NetBird primaire nameserver](../findings/netbird-primary-nameserver.nl.md).

---

## Eindverificatie

**Testdomein:** `testentry.rpz.urlhaus.abuse.ch` — een permanente testvermelding in de URLhaus RPZ-feed.

**Onderscheid tussen RPZ en reguliere NXDOMAIN:** Een RPZ-blokkering heeft de `aa`-vlag (authoritative answer). Reguliere internet NXDOMAIN heeft deze niet.

**Testpunt 1 — pop01 lokaal:**

```bash
drill @127.0.0.1 testentry.rpz.urlhaus.abuse.ch
# Verwacht: rcode: NXDOMAIN, flags: aa
```

**Testpunt 2 — mobile01 via NetBird:**

```powershell
nslookup testentry.rpz.urlhaus.abuse.ch
# Verwacht: Non-existent domain
```

RPZ-log moet bron-IP `100.70.95.98` (mobile01) tonen.

**Testpunt 3 — dc01 via DC-LAN:**

```bash
drill @10.0.0.1 testentry.rpz.urlhaus.abuse.ch
# Verwacht: rcode: NXDOMAIN, flags: aa
```

**Controletest — normaal domein werkt:**

```bash
drill @127.0.0.1 google.com
# Verwacht: rcode: NOERROR, geen aa-vlag, geldig A-record
```

---

## Bekende openstaande punten

- **ioc2rpz GUI JS-fix is niet persistent** — opnieuw toepassen na containerrebuilds
- **DNS NOTIFY bereikt BIND niet direct** — ioc2rpz stuurt NOTIFY naar poort 53, BIND luistert op 53530. Feeds worden bijgewerkt op het poll-interval van 3600s. Handmatige trigger: `rndc -p 953 retransfer threat-intel.rpz.sase`

---

## Checklist

- [ ] ioc2rpz-containers draaien op mgmt01
- [ ] GUI toegankelijk en inloggen werkt
- [ ] 71.767+ RPZ-records geladen
- [ ] BIND-zonetransfer succesvol afgerond
- [ ] Unbound RPZ actief (4 modules geladen)
- [ ] NetBird primaire nameserver geconfigureerd (lege match-domeinen)
- [ ] pop01 lokaal: `testentry.rpz.urlhaus.abuse.ch` → NXDOMAIN + aa
- [ ] mobile01: zelfde test → NXDOMAIN via NetBird DNS-relay
- [ ] dc01: zelfde test → NXDOMAIN via DC-LAN
- [ ] Normale domeinen resolven nog steeds

---

## Gerelateerd

- [Component: ioc2rpz](../components/ioc2rpz.nl.md)
- [Concept: RPZ](../concepts/rpz.nl.md)
- [Beslissing: ioc2rpz vs Unbound native](../decisions/ioc2rpz-vs-unbound-native.nl.md)
- [Beslissing: BIND als TSIG-intermediair](../decisions/bind-tsig-intermediary.nl.md)
- [Finding: Unbound geen TSIG](../findings/unbound-no-tsig.nl.md)
- [Finding: Unbound configuratiepad](../findings/unbound-config-path.nl.md)
- [Finding: ioc2rpz GUI JS-fout](../findings/ioc2rpz-gui-js-bug.nl.md)
- [Finding: iptables FORWARD-volgorde](../findings/iptables-forward-ordering.nl.md)
- [Finding: NetBird primaire nameserver](../findings/netbird-primary-nameserver.nl.md)

---
title: "ioc2rpz + BIND + Unbound — DNS-bedreigingsinformatie"
tags: [ioc2rpz, dns, rpz, bind, unbound, docker, opnsense, sase, network]
---

# ioc2rpz + BIND + Unbound — DNS-bedreigingsinformatie

**Rol:** Proactieve DNS-blokkering van bekende kwaadaardige domeinen. ioc2rpz aggregeert bedreigingsinformatie-feeds (URLhaus, ThreatFox) in een RPZ-zone; BIND fungeert als TSIG-geauthenticeerde tussenpersoon; Unbound dwingt het RPZ-beleid af voor alle DNS-query's van BYOD-clients en DC-LAN-knooppunten.  
**Status:** ✅ Volledig operationeel — 71.767 RPZ-records uit twee feeds, gevalideerd vanaf drie testpunten.  
**Configuratielocaties:**
- ioc2rpz: `/opt/ioc2rpz/` op mgmt01 (Docker)
- BIND: `/usr/local/etc/namedb/` op pop01 (`os-bind`-plugin)
- Unbound: `/usr/local/etc/unbound.opnsense.d/rpz.conf` op pop01

---

## Werking in deze stack

DNS-bedreigingsinformatie onderschept zo vroeg mogelijk in de verbindingslevenscyclus — bij naamresolutie. Als een BYOD-client de hostnaam van een bekende malware-C2-server probeert op te zoeken, retourneert Unbound NXDOMAIN voordat er een TCP-verbinding tot stand kan komen, ongeacht poort of protocol.

**Driecomponentenketen:**

```
URLhaus + ThreatFox (Abuse.ch)
    ↓ HTTP-pull door ioc2rpz
ioc2rpz (mgmt01 Docker, 192.168.122.23:53)
    ↓ TSIG-geauthenticeerde AXFR (tkey_rpz_transfer, hmac-sha256)
BIND 9.20 (pop01, os-bind, 127.0.0.1:53530, secundaire zone)
    ↓ niet-geauthenticeerde AXFR (loopback)
Unbound (pop01, poort 53, respip-module)
    ↓ RPZ-afgedwongen responses
BYOD-clients + dc01
```

ioc2rpz is een **zonebron**, geen resolver. Clients vragen ioc2rpz nooit direct. Ze vragen Unbound, dat de RPZ-zone in het geheugen heeft. Geen extra latentie per query.

BIND bestaat uitsluitend omdat Unbound 1.24.2 TSIG voor zonetransfers niet ondersteunt — een ontbrekende functie gedocumenteerd in GitHub NLnetLabs/unbound issue #336 (open sinds oktober 2020). BIND verwerkt de TSIG-geauthenticeerde AXFR van ioc2rpz en presenteert de zone aan Unbound via loopback zonder authenticatie. Zie [Beslissing: BIND als TSIG-tussenpersoon](../decisions/bind-tsig-intermediary.md).

**NetBird primaire naamserver:** Om RPZ BYOD-clients op externe domeinen te beschermen, moeten alle DNS-query's via pop01 Unbound verlopen. De primaire naamserverinstelling van NetBird (overeenkomende domeinen: leeg) bereikt dit. Zonder dit gebruiken clients de standaard-DNS van hun adapter voor niet-interne domeinen, waardoor Unbound volledig wordt omzeild. Zie [Bevinding: NetBird primaire naamserver](../findings/netbird-primary-nameserver.md).

---

## Configuratie

### ioc2rpz Docker-implementatie

Locatie: `/opt/ioc2rpz/docker-compose.yml` op mgmt01.

Poortbindingen specifiek op `192.168.122.23` (systemd-resolved bezit `127.0.0.53:53`, NetBird DNS-relay bezit `100.70.135.241:53`):

```yaml
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
```

Vervang het meegeleverde SSL-certificaat (verlopen juli 2022) door een nieuw zelfondertekend certificaat vóór het opstarten.

### ioc2rpz GUI-configuratie

Toegang via `https://ioc2rpz.sandbox.local` (door Caddy geproxied). Na de eerste aanmelding configureren:

**Server:** Publiek IP `192.168.122.23`, NS `ns1.ioc2rpz.local`

**TSIG-sleutels:**
- `tkey_mgmt_1` — beheersleutel, hmac-md5
- `tkey_rpz_transfer` — zonetransfersleutel, hmac-sha256

**Bronnen:**
- `urlhaus_rpz` — `https://urlhaus.abuse.ch/downloads/rpz/`
- `threatfox_rpz` — `https://threatfox.abuse.ch/downloads/threatfox.rpz` (let op: niet `/downloads/rpz/`)

**RPZ-zone:**
- Naam: `threat-intel.rpz.sase`
- Actie: nxdomain, Wildcards: true
- TSIG-sleutel: `tkey_rpz_transfer`
- Notify: `192.168.122.13` (pop01)

Controleer na "Publiceren" of het zonelog ~71.000+ indicatoren toont en "DNS Notify sent".

**ioc2rpz.gui JavaScript-bug:** Het aanmeldformulier mist een `e.preventDefault()` in de `signIn`-functie, waardoor de browser native indient terwijl axios ook post — de native inzending herlaadt de pagina en breekt het axios-verzoek af. Pas de correctie eenmalig toe na containerstart:

```bash
docker exec ioc2rpz-ioc2rpz-gui-1 sed -i \
  "s|signIn: function(e){ //|signIn: function(e){ e.preventDefault(); //|" \
  /opt/ioc2rpz.gui/www/js/io2auth.js
```

Zie [Bevinding: ioc2rpz GUI JS-bug](../findings/ioc2rpz-gui-js-bug.md).

### BIND (os-bind) op pop01

Plugin installeren: Systeem → Firmware → Plugins → `os-bind`.

**ACL:** Maak `loopback_only` (`127.0.0.1/32`) aan vóór het configureren van algemene instellingen.

**Algemene instellingen:**
- Luister-IP's: `127.0.0.1`, Luisterpoort: `53530`
- DNS-doorstuurservers: (leeg), Recursie: geen, Transfer toestaan: `loopback_only`

**Secundaire zone:**
- Zonenaam: `threat-intel.rpz.sase`
- Type: Secundair
- Primair IP: `192.168.122.23`
- Transfersleutel: `tkey_rpz_transfer`, Algoritme: hmac-sha256
- Query toestaan: `loopback_only` (expliciet instellen — niet leeg laten)

De gegenereerde `named.conf` gebruikt moderne inline TSIG-syntaxis:
```
primaries { 192.168.122.23 key "tkey_rpz_transfer"; };
```

### Unbound RPZ op pop01

Persistente configuratielocatie (overleeft OPNsense-herstarts — het juiste pad):
```
/usr/local/etc/unbound.opnsense.d/rpz.conf
```

⚠️ Gebruik **niet** `/var/unbound/etc/rpz.conf` — OPNsense regenereert de chroot-map bij elke Unbound-herstart en verwijdert alle bestanden die daar zijn geplaatst. Zie [Bevinding: Unbound-configuratiepad](../findings/unbound-config-path.md).

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

**`python` moet in module-config blijven** — OPNsense gebruikt `unbound-dnsbl/dnsbl_module.py` voor de ingebouwde DNS-blokkeerlijstfunctie. Het verwijderen van `python` uit module-config breekt dit.

**`configctl unbound check` fout-positief** — de controle heeft een vaste whitelist van "bekende" modulecombinaties. `respip python validator iterator` staat niet op de lijst maar werkt wel tijdens runtime. Negeer de fout en ga door met de herstart.

Toepassen:
```bash
configctl unbound restart
```

Controleer of vier modules geladen zijn:
```
[notice: init module 0: respip
[notice: init module 1: python
[notice: init module 2: validator
[notice: init module 3: iterator
```

### Validatie

Testdomein: `testentry.rpz.urlhaus.abuse.ch` — altijd aanwezig in de URLhaus-feed, ontworpen voor RPZ-validatie.

```bash
# pop01 lokaal
drill @127.0.0.1 testentry.rpz.urlhaus.abuse.ch
# Verwacht: rcode: NXDOMAIN, flags: aa (gezaghebbend — bevestigt lokale RPZ, niet internet-NXDOMAIN)

# mobile01 via NetBird
nslookup testentry.rpz.urlhaus.abuse.ch
# Verwacht: Niet-bestaand domein (via 100.70.255.254 NetBird DNS-relay)
```

RPZ-log op pop01 bevestigt bron:
```
rpz: applied [ioc2rpz-threat-intel] testentry.rpz.urlhaus.abuse.ch. rpz-nxdomain <client-ip>
```

---

## Integratiepunten

| Component | Richting | Wat |
|-----------|----------|-----|
| [NetBird](netbird.md) | afhankelijkheid | Primaire naamserverinstelling routeert alle client-DNS via pop01 Unbound |
| [Caddy](caddy.md) | → browser | Caddy proxiet ioc2rpz GUI op `https://ioc2rpz.sandbox.local` |
| [GNS3/Topologie](gns3.md) | afhankelijkheid | ioc2rpz→BIND-zonetransfer gebruikt 192.168.122.x (WAN-segment); iptables FORWARD-regels moeten `-I FORWARD 1` gebruiken |
| [Suricata](suricata.md) | aanvullend | Abuse.ch URLhaus-feeds verschijnen in beide; Suricata detecteert verbindingen met C2-IP's, RPZ voorkomt DNS-resolutie van C2-domeinen |

---

## Bekende problemen / valkuilen

**iptables FORWARD-regelvolgorde** — gebruik bij het toevoegen van poortdoorsturingen voor GUI-toegang op de GNS3-host altijd `-I FORWARD 1`. Toevoegen (`-A`) plaatst de regel na de REJECT-keten van libvirt. Zie [Bevinding: iptables FORWARD-volgorde](../findings/iptables-forward-ordering.md).

**ioc2rpz.gui JS-aanmeldbug** — upstream-bug in `io2auth.js`. Pas de sed-correctie toe na elke containeropbouw. Zie [Bevinding: ioc2rpz GUI JS-bug](../findings/ioc2rpz-gui-js-bug.md).

**DNS NOTIFY bereikt BIND niet** — ioc2rpz stuurt NOTIFY naar `192.168.122.13:53` (Unbound-poort), niet `53530` (BIND-poort). BIND ontdekt updates alleen bij de volgende SOA-poll (3600 s). Handmatige activering: `rndc retransfer threat-intel.rpz.sase`. Voeg in productie een `pf rdr`-regel toe die poort-53 NOTIFY van `192.168.122.23` doorstuurt naar poort 53530.

**OPNsense Unbound WebUI toont "handmatige overschrijvingen"-waarschuwing** — dit is verwacht en onschadelijk. Het rpz.conf overschrijft module-config, wat OPNsense detecteert. Documenteer dit in operationele runbooks zodat beheerders hier niet door verrast worden.

**TSIG-sleutel moet gekoppeld zijn aan RPZ-zone in ioc2rpz** — de sleutel moet expliciet worden geselecteerd in GUI → Configuratie → RPZ-zones. Als leeg gelaten (`tkeys: [""]`), ontvangt BIND TSIG-fouten (`tsig indicates error`) bij elke AXFR-poging.

---

## Gerelateerd

- [Architectuuroverzicht](../overview/architecture.md)
- [Concept: RPZ / DNS-bedreigingsinformatie](../concepts/rpz.md)
- [Beslissing: ioc2rpz vs. Unbound native](../decisions/ioc2rpz-vs-unbound-native.md)
- [Beslissing: BIND als TSIG-tussenpersoon](../decisions/bind-tsig-intermediary.md)
- [Bevinding: Unbound geen TSIG](../findings/unbound-no-tsig.md)
- [Bevinding: Unbound-configuratiepad](../findings/unbound-config-path.md)
- [Bevinding: ioc2rpz GUI JS-bug](../findings/ioc2rpz-gui-js-bug.md)
- [Bevinding: iptables FORWARD-volgorde](../findings/iptables-forward-ordering.md)
- [Bevinding: NetBird primaire naamserver](../findings/netbird-primary-nameserver.md)

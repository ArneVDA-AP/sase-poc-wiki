---
title: "Runbook: RITA → RPZ integratie, geautomatiseerde beacon-blokkering"
tags: [runbook, rita, rpz, ioc2rpz, dns, beaconing]
---

# Runbook: RITA → RPZ integratie, geautomatiseerde beacon-blokkering

**Bron:** `Verslag07` Deel II (mei 2026)
**Node(s):** mgmt01 (`192.168.122.20`) extractiescript + ioc2rpz; pop01 (`192.168.122.11`) BIND + Unbound
**Vereisten:** [Runbook 14: Zeek & RITA](14-zeek-rita.nl.md) (RITA produceert beacon scores) · [Runbook 06: DNS Threat Intelligence](06-dns-threat-intel.nl.md) (ioc2rpz → BIND → Unbound keten)
**Status:** ✅ PoC-gevalideerd op de parallelle stack, sandbox-integratie pending

> **Scope.** Gebouwd op de **parallelle stack**. De RITA → RPZ pipeline voedt dezelfde ioc2rpz → BIND →
> Unbound keten die de sandbox gebruikt, maar als parallel-stack-experiment: hier bindt ioc2rpz
> `192.168.122.20:53` en is pop01 `192.168.122.11`. De eigen RPZ van de sandbox (ioc2rpz op
> `192.168.122.23`, pop01 op `192.168.122.13`) blijft de source of truth. Runbook 06 documenteert
> de enforcement-keten volledig; deze runbook voegt er alleen RITA als derde bron bovenop toe.

---

## Wat dit opbouwt

RITA's beacon-detector wordt een automatische DNS-blokkering. Een cron job leest
`rita view --beacon`, filtert het terug tot high-confidence domeinen, en zet ze in een bestand dat
ioc2rpz al als feed-source leest. Vanaf daar neemt de bestaande keten het over: ioc2rpz merget alle
drie de feeds in `threat-intel.rpz.sase`, BIND transfereert de zone over TSIG, en Unbound enforcet
NXDOMAIN.

```
RITA (--beacon, ClickHouse) ─► rita_beacon_feed.sh (cron 15 min, score ≥ 0.8, minus whitelist)
   ─► /opt/ioc2rpz/cfg/rita_beacons.txt
   ─► ioc2rpz  (merge met URLhaus + ThreatFox → threat-intel.rpz.sase, nieuw serial)
   ─► BIND  (TSIG-AXFR bij SOA-poll)
   ─► Unbound  (RPZ-check → NXDOMAIN + aa)
```

Zie [Beslissing: RITA als derde dynamische RPZ-feed](../decisions/rita-rpz-automation.nl.md) voor
waarom een file-feed in ioc2rpz is gekozen boven een directe Unbound-schrijf of handmatige review.

---

## Vereistenchecklist

- [ ] RITA `sase_poc` database gevuld en `rita view --beacon` geeft rijen terug (Runbook 14)
- [ ] ioc2rpz draait op mgmt01 met de `threat-intel.rpz.sase` zone live (Runbook 06)
- [ ] BIND secondary + Unbound RPZ op pop01 enforcen al URLhaus/ThreatFox (Runbook 06)
- [ ] Root/sudo op mgmt01

---

## Stap 1: bevestig RITA's CSV-output formaat

Het extractiescript parset `rita view -o` CSV op kolompositie, dus bevestig eerst de indeling:

```bash
sudo rita view -o sase_poc --beacon 2>/dev/null | head -3
```

Header:

```
Severity,Source IP,Destination IP,FQDN,Beacon Score,Strobe,Total Duration,...
```

De relevante kolommen: **Severity = kolom 1, FQDN = kolom 4, Beacon Score = kolom 5**.

> De eerste twee regels van `rita view -o` zijn een "Viewing database: ..."-banner en de
> CSV-header. Beide moeten overgeslagen worden (`tail -n +3`).

---

## Stap 2: bouw de whitelist

Gedragsdetectie flagt legitieme periodieke services. Zonder whitelist zou de RPZ NetBird (100%
beacon score) en Ubuntu NTP (99%) blokkeren. De whitelist is het eerste controle-punt voor false
positives. Voeg hier een domein toe op het moment dat een bekend-goede service als beacon
opduikt.

```bash
sudo tee /opt/ioc2rpz/cfg/rita_whitelist.txt << 'EOF'
# NetBird overlay infrastructure
pkgs.netbird.io
netbird.io
relay.netbird.io
# Ubuntu/Canonical NTP en updates
ubuntu.com
canonical.com
ntp.ubuntu.com
# DNS infrastructure
abuse.ch
ioc2rpz.local
sase.local
# OPNsense/FreeBSD updates
opnsense.org
freebsd.org
pkg.freebsd.org
# Zeek/RITA eigen telemetrie
zeek.org
activecountermeasures.com
# Cloud services in gebruik
github.com
microsoft.com
login.microsoftonline.com
# Google DNS
dns.google
EOF
```

> **Gotcha: `sudo cat > bestand` werkt niet op Ubuntu.** De redirect draait vóór `sudo`. Gebruik
> `sudo tee`.

---

## Stap 3: schrijf het extractiescript

```bash
sudo mkdir -p /opt/scripts
sudo tee /opt/scripts/rita_beacon_feed.sh << 'SCRIPT'
#!/bin/bash
DATABASE="sase_poc"
OUTPUT="/opt/ioc2rpz/cfg/rita_beacons.txt"
WHITELIST="/opt/ioc2rpz/cfg/rita_whitelist.txt"
TMPFILE="/tmp/rita_beacons_staging.txt"
LOGFILE="/var/log/rita_beacon_feed.log"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" >> "$LOGFILE"; }
log "START — extracting beacons from $DATABASE"

sudo rita view -o "$DATABASE" --beacon 2>/dev/null \
  | tail -n +3 \
  | awk -F',' '$5 >= 0.8 && $4 != "" { print $4 }' \
  | grep -E '^[a-zA-Z0-9][a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$' \
  | grep -ivf "$WHITELIST" \
  | sort -u \
  > "$TMPFILE" || true

COUNT=0
if [ -f "$TMPFILE" ]; then
    COUNT=$(wc -l < "$TMPFILE")
fi

if [ "$COUNT" -gt 0 ]; then
    mv "$TMPFILE" "$OUTPUT"
    log "OK — $COUNT domeinen geschreven naar $OUTPUT"
else
    touch "$OUTPUT"
    rm -f "$TMPFILE"
    log "OK — geen beacons boven threshold, leeg bestand"
fi
log "END"
SCRIPT

sudo chmod +x /opt/scripts/rita_beacon_feed.sh
```

> **Gotcha: geen `set -euo pipefail`.** Een lege `grep`-output (geen matches) geeft exit 1, wat het
> script zou afbreken. De `|| true` aan het einde van de pipeline vangt dat op.

> **Waarom score, niet severity.** Het script filtert op beacon score ≥ 0.8 (kolom 5), niet op
> RITA's severity. Severity is conservatief: `httpbin.org` met 333 connecties en score 0.629 kreeg
> slechts "Medium". De score is de betrouwbaardere trigger. Zie
> [Beslissing: RITA als derde dynamische RPZ-feed](../decisions/rita-rpz-automation.nl.md).

---

## Stap 4: voeg de `rita_beacons`-source toe aan ioc2rpz

Voeg in ioc2rpz' config (`/opt/ioc2rpz/cfg/ioc2rpz.conf`) een `file:`-source toe en zet die in de
sources van de zone naast `urlhaus_rpz` en `threatfox_rpz`:

```erlang
{source,{"rita_beacons","file:/opt/ioc2rpz/cfg/rita_beacons.txt","[:AXFR:]","^([A-Za-z0-9][A-Za-z0-9\-\._]+)$","",0,900,0}}.

{rpz,{"threat-intel.rpz.sase",300,3600,2592000,7200,"true","true","nxdomain",["tkey_rpz_transfer"],"fqdn",3600,86400,["urlhaus_rpz","threatfox_rpz","rita_beacons"],["192.168.122.11"],[]}}.
```

> **Gotcha: herschrijf het hele configbestand, gebruik geen `sed`.** Regels toevoegen aan de
> Erlang-config met `sed` matchte meerdere regels en verdubbelde alles. Schrijf het volledige
> bestand met `sudo tee`. (Dit is dezelfde harde les als de GUI-export die niet betrouwbaar
> schreef: schrijf de config direct.)

> **Let op de SOA refresh = 300.** Het eerste getal na de zonenaam is de SOA refresh in seconden.
> Het is verlaagd van `86400` (24 u) naar `300` (5 min) zodat Unbound de zone binnen vijf minuten na
> een update opnieuw ophaalt. Het NOTIFY-doel `192.168.122.11` is pop01 op de parallelle stack.

Toepassen en verifiëren:

```bash
sudo docker restart ioc2rpz-ioc2rpz-1
sleep 30
sudo docker logs ioc2rpz-ioc2rpz-1 2>&1 | grep "threat-intel"
# Verwacht: urlhaus_rpz ~577, threatfox_rpz ~61130, zone "threat-intel.rpz.sase" updated
```

---

## Stap 5: plan de cron job

```bash
sudo tee /etc/cron.d/rita-beacon-feed << 'EOF'
*/15 * * * * root /opt/scripts/rita_beacon_feed.sh
EOF
```

Het script draait elke 15 minuten, regenereert `rita_beacons.txt`, en ioc2rpz pikt het bestand op
bij de volgende zone-update.

> **Open punt: RITA-import is nog handmatig.** Het extractiescript herleest alleen data die RITA al
> heeft geïmporteerd. Als `rita import` niet ook op een schema draait, zitten verse Zeek-logs nog
> niet in de analyse. Een conceptuele hourly import (`rita import -l /opt/zeek/logs/$(date
> +\%Y-\%m-\%d)/ --database sase_poc --rolling`) was genoteerd maar nog niet ingericht ten tijde van
> het verslag.

---

## Stap 6: end-to-end validatie

**Gecontroleerde test (een geplant domein loopt de hele keten):**

```bash
# Op mgmt01: plaats een domein handmatig
echo "fake-c2-beacon-test.example.org" | sudo tee -a /opt/ioc2rpz/cfg/rita_beacons.txt
```

1. ioc2rpz laadt het als één indicator en verhoogt het zone-serial.
2. BIND transfereert de zone (nieuw serial) over TSIG.
3. Unbound blokkeert het: `NXDOMAIN + aa`.

```bash
# Op pop01 (shell):
drill @127.0.0.1 fake-c2-beacon-test.example.org
# Verwacht: rcode: NXDOMAIN, flags: aa
```

**Live test, een echte beacon opgevangen door RITA:** `grafana.com` (de lokale monitoring-stack
die naar buiten pollt) werd door RITA als Critical gescoord, geëxtraheerd naar `rita_beacons.txt`,
en op dezelfde manier geblokkeerd: NXDOMAIN terwijl het op noch URLhaus noch ThreatFox staat.

```bash
cat /opt/ioc2rpz/cfg/rita_beacons.txt        # → grafana.com
# Op een NetBird-verbonden Windows-client:
#   nslookup grafana.com → Non-existent domain
```

> **Gotcha: Unbound cachet het zone-bestand en ververst niet altijd vanzelf.** De zone staat op
> `/var/unbound/threat-intel.rpz.sase.zone` en wordt pas overschreven bij een nieuwe AXFR. Als een
> update niet propageert, forceer het op pop01:
> ```bash
> configctl bind stop
> rm /usr/local/etc/namedb/secondary/threat-intel.rpz.sase.db
> configctl bind start
> sleep 15
> rm /var/unbound/threat-intel.rpz.sase.zone
> configctl unbound restart
> ```
> Het verlagen van de SOA refresh naar 300 s (Stap 4) zou dit automatisch moeten maken, maar
> volledige auto-propagatie binnen vijf minuten was ten tijde van het verslag nog niet definitief
> geverifieerd.

---

## Bekende open punten

- **Auto-propagatie binnen 5 min niet volledig bevestigd.** SOA refresh is 300 s; of Unbound
  ververst zonder handmatige `rm zone + restart` werd nog gevalideerd via een
  `windows-update-service.net` beacon-test.
- **RITA-import nog niet op cron.** Alleen het extractiescript is gepland; hourly `rita import`
  blijft handmatig.
- **Scoring is venster-gevoelig.** Een rolling import verwerkt het meest recente uur-chunk, dus een
  domein dat alleen in een eerder venster beaconde kan laag scoren bij een latere import. Laat een
  beacon lang genoeg binnen het huidige venster lopen.

---

## Checklist

- [ ] `rita view -o sase_poc --beacon` CSV bevestigd (FQDN kolom 4, score kolom 5)
- [ ] `rita_whitelist.txt` geschreven (NetBird, NTP, infra-domeinen)
- [ ] `rita_beacon_feed.sh` uitvoerbaar, filtert score ≥ 0.8, `|| true` aanwezig
- [ ] `rita_beacons` `file:`-source toegevoegd aan ioc2rpz; zone bevat alle drie de sources
- [ ] ioc2rpz-restart toont de zone geüpdatet met alle drie de feeds
- [ ] cron `/etc/cron.d/rita-beacon-feed` draait elke 15 min
- [ ] Geplant `fake-c2-beacon-test.example.org` → NXDOMAIN + aa op pop01
- [ ] Live `grafana.com` (RITA Critical) → NXDOMAIN op een client

---

## Gerelateerd

- [Component: RITA](../components/rita.nl.md)
- [Component: ioc2rpz + BIND + Unbound](../components/ioc2rpz.nl.md)
- [Concept: Behavioral analysis](../concepts/behavioral-analysis.nl.md)
- [Beslissing: RITA als derde dynamische RPZ-feed](../decisions/rita-rpz-automation.nl.md)
- [Beslissing: ioc2rpz vs Unbound native RPZ](../decisions/ioc2rpz-vs-unbound-native.nl.md)
- [Runbook 14: Zeek & RITA](14-zeek-rita.nl.md)
- [Runbook 06: DNS Threat Intelligence](06-dns-threat-intel.nl.md)

---
title: "Runbook: RITA → RPZ Integration — Automated Beacon Blocking"
tags: [runbook, rita, rpz, ioc2rpz, dns, beaconing]
---

# Runbook: RITA → RPZ Integration — Automated Beacon Blocking

**Source:** `Verslag07` Deel II (May 2026)
**Node(s):** mgmt01 (`192.168.122.20`) extraction script + ioc2rpz; pop01 (`192.168.122.11`) BIND + Unbound
**Prerequisites:** [Runbook 14: Zeek & RITA](14-zeek-rita.md) (RITA producing beacon scores) · [Runbook 06: DNS Threat Intelligence](06-dns-threat-intel.md) (ioc2rpz → BIND → Unbound chain)
**Status:** ✅ PoC-validated on the parallel stack — sandbox integration pending

> **Scope.** Built on the **parallel stack**. The RITA → RPZ pipeline feeds the same ioc2rpz → BIND → Unbound
> chain the sandbox uses, but as a parallel-stack experiment: here ioc2rpz binds
> `192.168.122.20:53` and pop01 is `192.168.122.11`. The sandbox's own RPZ (ioc2rpz on
> `192.168.122.23`, pop01 on `192.168.122.13`) remains the source of truth. Runbook 06 documents
> the enforcement chain in full; this runbook only adds RITA as a third source on top of it.

---

## What this builds

RITA's beacon detector becomes an automatic DNS block. A cron job reads `rita view --beacon`,
filters it down to high-confidence domains, and drops them into a file that ioc2rpz already reads
as a feed source. From there the existing chain takes over: ioc2rpz merges all three feeds into
`threat-intel.rpz.sase`, BIND transfers the zone over TSIG, and Unbound enforces NXDOMAIN.

```
RITA (--beacon, ClickHouse) ─► rita_beacon_feed.sh (cron 15 min, score ≥ 0.8, minus whitelist)
   ─► /opt/ioc2rpz/cfg/rita_beacons.txt
   ─► ioc2rpz  (merge with URLhaus + ThreatFox → threat-intel.rpz.sase, new serial)
   ─► BIND  (TSIG AXFR on SOA poll)
   ─► Unbound  (RPZ check → NXDOMAIN + aa)
```

See [Decision: RITA as a third dynamic RPZ feed](../decisions/rita-rpz-automation.md) for why a
file feed into ioc2rpz was chosen over a direct Unbound write or manual review.

---

## Prerequisites checklist

- [ ] RITA `sase_poc` database populated and `rita view --beacon` returns rows (Runbook 14)
- [ ] ioc2rpz running on mgmt01 with the `threat-intel.rpz.sase` zone live (Runbook 06)
- [ ] BIND secondary + Unbound RPZ on pop01 already enforcing URLhaus/ThreatFox (Runbook 06)
- [ ] Root/sudo on mgmt01

---

## Step 1: Confirm RITA's CSV output format

The extraction script parses `rita view -o` CSV by column position, so confirm the layout first:

```bash
sudo rita view -o sase_poc --beacon 2>/dev/null | head -3
```

Header:

```
Severity,Source IP,Destination IP,FQDN,Beacon Score,Strobe,Total Duration,...
```

The columns that matter: **Severity = col 1, FQDN = col 4, Beacon Score = col 5**.

> The first two lines of `rita view -o` are a "Viewing database: ..." banner and the CSV header.
> Both must be skipped (`tail -n +3`).

---

## Step 2: Build the whitelist

Behavioral detection flags legitimate periodic services. Without a whitelist the RPZ would block
NetBird (100% beacon score) and Ubuntu NTP (99%). The whitelist is the first control point for
false positives — add a domain here the moment a known-good service shows up as a beacon.

```bash
sudo tee /opt/ioc2rpz/cfg/rita_whitelist.txt << 'EOF'
# NetBird overlay infrastructure
pkgs.netbird.io
netbird.io
relay.netbird.io
# Ubuntu/Canonical NTP and updates
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
# Zeek/RITA own telemetry
zeek.org
activecountermeasures.com
# Cloud services in use
github.com
microsoft.com
login.microsoftonline.com
# Google DNS
dns.google
EOF
```

> **Gotcha: `sudo cat > file` does not work on Ubuntu** — the redirect runs before `sudo`. Use
> `sudo tee`.

---

## Step 3: Write the extraction script

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
    log "OK — $COUNT domains written to $OUTPUT"
else
    touch "$OUTPUT"
    rm -f "$TMPFILE"
    log "OK — no beacons above threshold, empty file"
fi
log "END"
SCRIPT

sudo chmod +x /opt/scripts/rita_beacon_feed.sh
```

> **Gotcha: no `set -euo pipefail`.** An empty `grep` result (no matches) exits 1, which would
> abort the script. The `|| true` at the end of the pipeline absorbs that.

> **Why score, not severity.** The script filters on beacon score ≥ 0.8 (col 5), not RITA's
> severity. Severity is conservative: `httpbin.org` with 333 connections and score 0.629 rated
> only "Medium". The score is the more reliable trigger. See
> [Decision: RITA as a third dynamic RPZ feed](../decisions/rita-rpz-automation.md).

---

## Step 4: Add the `rita_beacons` source to ioc2rpz

In ioc2rpz's config (`/opt/ioc2rpz/cfg/ioc2rpz.conf`), add a `file:` source and list it in the
zone's sources alongside `urlhaus_rpz` and `threatfox_rpz`:

```erlang
{source,{"rita_beacons","file:/opt/ioc2rpz/cfg/rita_beacons.txt","[:AXFR:]","^([A-Za-z0-9][A-Za-z0-9\-\._]+)$","",0,900,0}}.

{rpz,{"threat-intel.rpz.sase",300,3600,2592000,7200,"true","true","nxdomain",["tkey_rpz_transfer"],"fqdn",3600,86400,["urlhaus_rpz","threatfox_rpz","rita_beacons"],["192.168.122.11"],[]}}.
```

> **Gotcha: rewrite the whole config file, do not `sed` it.** Adding lines to the Erlang config
> with `sed` matched multiple lines and duplicated everything. Write the full file with
> `sudo tee`. (This is the same hard lesson as the GUI export not writing reliably — author the
> config directly.)

> **Note the SOA refresh = 300.** The first number after the zone name is the SOA refresh in
> seconds. It was lowered from `86400` (24 h) to `300` (5 min) so Unbound re-pulls the zone within
> five minutes of an update. The NOTIFY target `192.168.122.11` is pop01 on the parallel stack.

Apply and verify:

```bash
sudo docker restart ioc2rpz-ioc2rpz-1
sleep 30
sudo docker logs ioc2rpz-ioc2rpz-1 2>&1 | grep "threat-intel"
# Expected: urlhaus_rpz ~577, threatfox_rpz ~61130, zone "threat-intel.rpz.sase" updated
```

---

## Step 5: Schedule the cron job

```bash
sudo tee /etc/cron.d/rita-beacon-feed << 'EOF'
*/15 * * * * root /opt/scripts/rita_beacon_feed.sh
EOF
```

The script runs every 15 minutes, regenerates `rita_beacons.txt`, and ioc2rpz picks up the file on
its next zone update.

> **Open point — RITA import is still manual.** The extraction script only re-reads data RITA has
> already imported. If `rita import` is not also running on a schedule, fresh Zeek logs are not in
> the analysis yet. A conceptual hourly import (`rita import -l /opt/zeek/logs/$(date +\%Y-\%m-\%d)/
> --database sase_poc --rolling`) was noted but not yet wired up at the time of the report.

---

## Step 6: End-to-end validation

**Controlled test — a planted domain walks the whole chain:**

```bash
# On mgmt01: place a domain by hand
echo "fake-c2-beacon-test.example.org" | sudo tee -a /opt/ioc2rpz/cfg/rita_beacons.txt
```

1. ioc2rpz loads it as one indicator and bumps the zone serial.
2. BIND transfers the zone (new serial) over TSIG.
3. Unbound blocks it: `NXDOMAIN + aa`.

```bash
# On pop01 (shell):
drill @127.0.0.1 fake-c2-beacon-test.example.org
# Expected: rcode: NXDOMAIN, flags: aa
```

**Live test — a real beacon caught by RITA:** `grafana.com` (the local monitoring stack polling
out) was scored Critical by RITA, extracted into `rita_beacons.txt`, and blocked the same way —
NXDOMAIN although it is on neither URLhaus nor ThreatFox.

```bash
cat /opt/ioc2rpz/cfg/rita_beacons.txt        # → grafana.com
# On a NetBird-connected Windows client:
#   nslookup grafana.com → Non-existent domain
```

> **Gotcha: Unbound caches the zone file and does not always refresh on its own.** The zone lives
> at `/var/unbound/threat-intel.rpz.sase.zone` and is only overwritten on a new AXFR. If an update
> does not propagate, force it on pop01:
> ```bash
> configctl bind stop
> rm /usr/local/etc/namedb/secondary/threat-intel.rpz.sase.db
> configctl bind start
> sleep 15
> rm /var/unbound/threat-intel.rpz.sase.zone
> configctl unbound restart
> ```
> Lowering the SOA refresh to 300 s (Step 4) should make this automatic, but full auto-propagation
> within five minutes was not yet definitively verified at the time of the report.

---

## Known open points

- **Auto-propagation within 5 min not fully confirmed.** SOA refresh is 300 s; whether Unbound
  refreshes without a manual `rm zone + restart` was still being validated via a
  `windows-update-service.net` beacon test.
- **RITA import not yet on a cron.** Only the extraction script is scheduled; hourly `rita import`
  remains manual.
- **Scoring is window-sensitive.** A rolling import processes the most recent hour chunk, so a
  domain that beaconed only in an earlier window can score low on a later import. Run a beacon long
  enough inside the current window.

---

## Checklist

- [ ] `rita view -o sase_poc --beacon` CSV confirmed (FQDN col 4, score col 5)
- [ ] `rita_whitelist.txt` written (NetBird, NTP, infra domains)
- [ ] `rita_beacon_feed.sh` executable, filters score ≥ 0.8, `|| true` present
- [ ] `rita_beacons` `file:` source added to ioc2rpz; zone lists all three sources
- [ ] ioc2rpz restart shows the zone updated with all three feeds
- [ ] cron `/etc/cron.d/rita-beacon-feed` runs every 15 min
- [ ] Planted `fake-c2-beacon-test.example.org` → NXDOMAIN + aa on pop01
- [ ] Live `grafana.com` (RITA Critical) → NXDOMAIN on a client

---

## Related

- [Component: RITA](../components/rita.md)
- [Component: ioc2rpz + BIND + Unbound](../components/ioc2rpz.md)
- [Concept: Behavioral analysis](../concepts/behavioral-analysis.md)
- [Decision: RITA as a third dynamic RPZ feed](../decisions/rita-rpz-automation.md)
- [Decision: ioc2rpz vs Unbound native RPZ](../decisions/ioc2rpz-vs-unbound-native.md)
- [Runbook 14: Zeek & RITA](14-zeek-rita.md)
- [Runbook 06: DNS Threat Intelligence](06-dns-threat-intel.md)

---
title: "ioc2rpz + BIND + Unbound — DNS Threat Intelligence"
tags: [ioc2rpz, dns, rpz, bind, unbound, docker, opnsense, sase, network]
---

# ioc2rpz + BIND + Unbound — DNS Threat Intelligence

**Role:** Proactive DNS-level blocking of known-malicious domains. ioc2rpz aggregates threat intelligence feeds (URLhaus, ThreatFox) into an RPZ zone; BIND acts as a TSIG-authenticated intermediary; Unbound enforces the RPZ policy for all DNS queries from BYOD clients and DC-LAN nodes.  
**Status:** ✅ Fully operational — 71 767 RPZ records from two feeds, validated from three test points.  
**Config locations:**
- ioc2rpz: `/opt/ioc2rpz/` on mgmt01 (Docker)
- BIND: `/usr/local/etc/namedb/` on pop01 (`os-bind` plugin)
- Unbound: `/usr/local/etc/unbound.opnsense.d/rpz.conf` on pop01

---

## How it works in this stack

DNS threat intelligence intercepts at the earliest possible point in the connection lifecycle — name resolution. If a BYOD client tries to look up the hostname of a known malware C2 server, Unbound returns NXDOMAIN before any TCP connection can be established, regardless of port or protocol.

**Three-component chain:**

```
URLhaus + ThreatFox (Abuse.ch)
    ↓ HTTP pull by ioc2rpz
ioc2rpz (mgmt01 Docker, 192.168.122.23:53)
    ↓ TSIG-authenticated AXFR (tkey_rpz_transfer, hmac-sha256)
BIND 9.20 (pop01, os-bind, 127.0.0.1:53530, secondary zone)
    ↓ unauthenticated AXFR (loopback)
Unbound (pop01, port 53, respip module)
    ↓ RPZ-enforced responses
BYOD clients + dc01
```

ioc2rpz is a **zone source**, not a resolver. Clients never query ioc2rpz directly. They query Unbound, which has the RPZ zone loaded in memory. No extra latency per query.

BIND exists solely because Unbound 1.24.2 does not support TSIG for zone transfers — a missing feature documented in GitHub NLnetLabs/unbound issue #336 (open since October 2020). BIND handles the TSIG-authenticated AXFR from ioc2rpz and presents the zone to Unbound via loopback without authentication. See [Decision: BIND as TSIG intermediary](../decisions/bind-tsig-intermediary.md).

**NetBird primary nameserver:** For RPZ to protect BYOD clients on external domains, all their DNS queries must route through pop01 Unbound. NetBird's primary nameserver setting (match domains: empty) achieves this. Without it, clients use their adapter's default DNS for non-internal domains, bypassing Unbound entirely. See [Finding: NetBird primary nameserver](../findings/netbird-primary-nameserver.md).

---

## Configuration

### ioc2rpz Docker deployment

Location: `/opt/ioc2rpz/docker-compose.yml` on mgmt01.

Port bindings on `192.168.122.23` specifically (systemd-resolved holds `127.0.0.53:53`, NetBird DNS relay holds `100.70.135.241:53`):

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

Replace the bundled SSL certificate (expired July 2022) with a new self-signed cert before starting.

### ioc2rpz GUI configuration

Access via `https://ioc2rpz.sandbox.local` (proxied by Caddy). After first login, configure:

**Server:** Public IP `192.168.122.23`, NS `ns1.ioc2rpz.local`, ACL: `127.0.0.1, 172.20.0.2` (GUI container), `172.20.0.1`

**TSIG keys:**
- `tkey_mgmt_1` — management key, hmac-md5
- `tkey_rpz_transfer` — zone transfer key, hmac-sha256

**Sources:**
- `urlhaus_rpz` — `https://urlhaus.abuse.ch/downloads/rpz/`
- `threatfox_rpz` — `https://threatfox.abuse.ch/downloads/threatfox.rpz` (note: not `/downloads/rpz/`)

**RPZ zone:**
- Name: `threat-intel.rpz.sase`
- Action: nxdomain, Wildcards: true
- TSIG key: `tkey_rpz_transfer`
- Notify: `192.168.122.13` (pop01)

After "Publish", verify the zone log shows ~71 000+ indicators and "DNS Notify sent".

**ioc2rpz.gui JavaScript bug:** The login form has a missing `e.preventDefault()` in the `signIn` function, causing the browser to submit natively while axios also posts — the native submit reloads the page, aborting the axios request. Apply the fix once after container start:

```bash
docker exec ioc2rpz-ioc2rpz-gui-1 sed -i \
  "s|signIn: function(e){ //|signIn: function(e){ e.preventDefault(); //|" \
  /opt/ioc2rpz.gui/www/js/io2auth.js
```

See [Finding: ioc2rpz GUI JS bug](../findings/ioc2rpz-gui-js-bug.md).

### BIND (os-bind) on pop01

Install plugin: System → Firmware → Plugins → `os-bind`.

**ACL:** Create `loopback_only` (`127.0.0.1/32`) before configuring General Settings.

**General Settings:**
- Listen IPs: `127.0.0.1`, Listen Port: `53530`
- DNS Forwarders: (empty), Recursion: none, Allow Transfer: `loopback_only`

**Secondary zone:**
- Zone name: `threat-intel.rpz.sase`
- Type: Secondary
- Primary IP: `192.168.122.23`
- Transfer key: `tkey_rpz_transfer`, Algorithm: hmac-sha256
- Allow Query: `loopback_only` (set explicitly — do not leave empty)

The generated `named.conf` uses modern inline TSIG syntax:
```
primaries { 192.168.122.23 key "tkey_rpz_transfer"; };
```

### Unbound RPZ on pop01

Persistent config location (survives OPNsense restarts — the correct path):
```
/usr/local/etc/unbound.opnsense.d/rpz.conf
```

⚠️ Do **not** use `/var/unbound/etc/rpz.conf` — OPNsense regenerates the chroot directory on every Unbound restart, deleting any files placed there. See [Finding: Unbound config path](../findings/unbound-config-path.md).

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

**`python` must stay in module-config** — OPNsense uses `unbound-dnsbl/dnsbl_module.py` for its built-in DNS blocklist feature. Removing `python` from module-config breaks this.

**`configctl unbound check` false positive** — the checker has a hardcoded whitelist of "known" module combinations. `respip python validator iterator` is not in the list but works at runtime. Ignore the error and proceed with restart. This is tracked in NLnetLabs/unbound issue #1373 (November 2025).

Apply:
```bash
configctl unbound restart
```

Verify four modules loaded:
```
[notice: init module 0: respip
[notice: init module 1: python
[notice: init module 2: validator
[notice: init module 3: iterator
```

### Validation

Test domain: `testentry.rpz.urlhaus.abuse.ch` — always present in the URLhaus feed, designed for RPZ validation.

```bash
# pop01 local
drill @127.0.0.1 testentry.rpz.urlhaus.abuse.ch
# Expected: rcode: NXDOMAIN, flags: aa (authoritative — confirms local RPZ, not internet NXDOMAIN)

# mobile01 via NetBird
nslookup testentry.rpz.urlhaus.abuse.ch
# Expected: Non-existent domain (via 100.70.255.254 NetBird DNS relay)
```

RPZ log on pop01 confirms source:
```
rpz: applied [ioc2rpz-threat-intel] testentry.rpz.urlhaus.abuse.ch. rpz-nxdomain <client-ip>
```

---

## Integration points

| Component | Direction | What |
|-----------|-----------|------|
| [NetBird](netbird.md) | dependency | Primary nameserver setting routes all client DNS through pop01 Unbound |
| [Caddy](caddy.md) | → browser | Caddy proxies ioc2rpz GUI at `https://ioc2rpz.sandbox.local` |
| [GNS3/Topology](gns3.md) | dependency | ioc2rpz→BIND zone transfer uses 192.168.122.x (WAN segment); iptables FORWARD rules must use `-I FORWARD 1` |
| [Suricata](suricata.md) | complementary | Abuse.ch URLhaus feeds appear in both; Suricata detects connections to C2 IPs, RPZ prevents DNS resolution of C2 domains |

---

## Known issues / gotchas

**iptables FORWARD rule ordering** — when adding port forwards for GUI access on the GNS3 host, always use `-I FORWARD 1`. Appending (`-A`) places the rule after libvirt's REJECT chain. See [Finding: iptables FORWARD ordering](../findings/iptables-forward-ordering.md).

**ioc2rpz.gui JS login bug** — upstream bug in `io2auth.js`. Apply the sed fix after each container rebuild. See [Finding: ioc2rpz GUI JS bug](../findings/ioc2rpz-gui-js-bug.md).

**DNS NOTIFY does not reach BIND** — ioc2rpz sends NOTIFY to `192.168.122.13:53` (Unbound port), not `53530` (BIND port). BIND discovers updates only at the next SOA poll (3600 s). Manual trigger: `rndc -p 953 retransfer threat-intel.rpz.sase`. In production, add a `pf rdr` rule redirecting port-53 NOTIFY from `192.168.122.23` to port 53530.

**BIND zone directory uses `secondary/`, not `slave/`** — modern BIND (9.20+) uses the naming convention `secondary` instead of the legacy `slave`. Zone files are stored at `/usr/local/etc/namedb/secondary/threat-intel.rpz.sase.db`. Documentation referencing `/slave/` is outdated.

**OPNsense Unbound WebUI shows "manual overwrites" warning** — this is expected and harmless. The rpz.conf overrides module-config, which OPNsense detects. Document this in operational runbooks so administrators are not surprised.

**TSIG key must be linked to RPZ zone in ioc2rpz** — the key must be explicitly selected in GUI → Configuration → RPZ Zones. If left empty (`tkeys: [""]`), BIND receives TSIG errors (`tsig indicates error`) for every AXFR attempt.

---

## Related

- [Architecture overview](../overview/architecture.md)
- [Concept: RPZ / DNS Threat Intelligence](../concepts/rpz.md)
- [Decision: ioc2rpz vs Unbound native](../decisions/ioc2rpz-vs-unbound-native.md)
- [Decision: BIND as TSIG intermediary](../decisions/bind-tsig-intermediary.md)
- [Finding: Unbound no TSIG](../findings/unbound-no-tsig.md)
- [Finding: Unbound config path](../findings/unbound-config-path.md)
- [Finding: ioc2rpz GUI JS bug](../findings/ioc2rpz-gui-js-bug.md)
- [Finding: iptables FORWARD ordering](../findings/iptables-forward-ordering.md)
- [Finding: NetBird primary nameserver](../findings/netbird-primary-nameserver.md)
- [Runbook: DNS Threat Intelligence](../runbooks/06-dns-threat-intel.md)

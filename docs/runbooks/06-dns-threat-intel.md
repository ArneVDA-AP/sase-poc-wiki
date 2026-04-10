---
title: "Runbook: DNS Threat Intelligence"
tags: [runbook, ioc2rpz, rpz, bind, unbound, dns]
---

# Runbook: DNS Threat Intelligence

**Source:** `raw/Doc4_DNS_Threat_Intelligence.md`
**Node(s):** mgmt01 Docker (ioc2rpz + GUI) + pop01 (BIND 9.20 + Unbound RPZ)
**Prerequisites:** [Runbook 02: ZTNA Overlay](02-ztna-overlay.md) completed (NetBird DNS relay operational)
**Status:** Operational — 71,767 RPZ records, three test points validated

---

## Prerequisites checklist

- [ ] NetBird overlay operational (Runbook 02)
- [ ] mgmt01 Docker daemon running
- [ ] pop01 Unbound resolving DNS queries
- [ ] Network path mgmt01 → pop01 over 192.168.122.x

---

## Step 1: Deploy ioc2rpz + GUI on mgmt01

**Resolve port conflicts first.** On mgmt01, port 53 is claimed by `systemd-resolved` and NetBird DNS relay. Bind ioc2rpz to the management IP `192.168.122.23`.

Create `/opt/ioc2rpz/docker-compose.yml`:

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

Renew the SSL certificate (the bundled one expired July 2022):

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

## Step 2: Configure iptables FORWARD rules for GUI access

> **Gotcha: Always use `-I FORWARD 1`, never `-A FORWARD`.** The FORWARD chain on the GNS3 host has a libvirt `LIBVIRT_FWI` chain at position ~22 that REJECTs new connections before appended rules are reached. You see 0 packets on your ACCEPT rules — the most hidden gotcha of the entire implementation.
> See [Finding: iptables FORWARD ordering](../findings/iptables-forward-ordering.md).

```bash
sudo iptables -I FORWARD 1 -d 192.168.122.23 -p tcp --dport 8080 -j ACCEPT
sudo iptables -I FORWARD 1 -d 192.168.122.23 -p tcp --dport 8444 -j ACCEPT
```

Persist with `netfilter-persistent`.

---

## Step 3: Fix ioc2rpz GUI JavaScript bug

Login fails with "Unknown error!!!" in the browser, while `curl` to `/io2auth.php/signin` returns `authSuccess`.

**Root cause:** The `signIn` function in `io2auth.js` is missing `e.preventDefault()`. The browser does a native form submit (synchronous page reload) that aborts the async axios POST.

```bash
sudo docker exec ioc2rpz-ioc2rpz-gui-1 sed -i \
  "s|signIn: function(e){ //|signIn: function(e){ e.preventDefault(); //|" \
  /opt/ioc2rpz.gui/www/js/io2auth.js
```

Hard refresh (Ctrl+Shift+R) after the fix. See [Finding: ioc2rpz GUI JS bug](../findings/ioc2rpz-gui-js-bug.md).

> **Gotcha: This patch is non-persistent.** It lives in the container layer. A `docker compose down && up` with a new image loses the patch. Re-apply after container rebuilds.

**Password requirements:** >7 chars + at least 1 digit, 1 lowercase, 1 uppercase, 1 special char. OR >15 chars.

**Caddy reverse proxy** for browser access via `ioc2rpz.sandbox.local` — add to the Caddyfile on mgmt01:

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

## Step 4: Configure ioc2rpz server, TSIG, feeds, and RPZ zone

Via the GUI after login:

**Server configuration:**
```
Name:      sase-poc-rpz
MGMT IP:   172.20.0.3 (Docker internal)
Public IP: 192.168.122.23
NS:        ns1.ioc2rpz.local
```

**TSIG Keys (create two):**
```
tkey_mgmt_1        — management key, hmac-md5
tkey_rpz_transfer  — zone transfer key, hmac-sha256
```

**Sources (Threat Feeds):**
```
urlhaus_rpz:   https://urlhaus.abuse.ch/downloads/rpz/
threatfox_rpz: https://threatfox.abuse.ch/downloads/threatfox.rpz
```

Note: the ThreatFox RPZ URL is `/downloads/threatfox.rpz`, NOT `/downloads/rpz/` (that gives 404).

**RPZ Zone:**
```
Name:       threat-intel.rpz.sase
Action:     nxdomain
Cache:      true
Wildcards:  true
TSIG key:   tkey_rpz_transfer
Sources:    urlhaus_rpz + threatfox_rpz
Notify:     192.168.122.13  (pop01)
AXFR time:  3600
```

> **Gotcha: TSIG key MUST be linked to the zone.** An unlinked key (empty `tkeys` field in the config) causes zone transfer failure with "tsig indicates error". Verify in the GUI that the zone shows the correct TSIG key, then click Publish.

After Publish, expected output:
```
Source: "urlhaus_rpz", got 450 indicators
Source: "threatfox_rpz", got 35475 indicators
Zone "threat-intel.rpz.sase" updated, serial ..., 71836 rules, 35918 indicators
```

---

## Step 5: Install BIND 9.20 on pop01

Via System → Firmware → Plugins: install `os-bind`. After installation: Services → BIND appears in menu.

> **Gotcha: Create the ACL before General Settings.** The ACL dropdown in General Settings only shows ACLs that already exist. If you configure General Settings first, the ACL dropdown is empty.

**ACL:** Services → BIND → Access Control List:
```
Name:     loopback_only
Networks: 127.0.0.1/32
```

Navigate away and back (GUI dropdown quirk).

**General Settings:**
```
Enable BIND Daemon: checked
Listen IPs:         127.0.0.1
Listen Port:        53530
DNS Forwarders:     empty
Recursion:          no ACL (off)
DNSSec Validation:  no
Allow Transfer:     loopback_only
```

**Secondary zone:** Services → BIND → Secondary Zones:
```
Zone name:          threat-intel.rpz.sase
Primary IP:         192.168.122.23
Primary Port:       53
Transfer Key:       tkey_rpz_transfer
Transfer Key Algo:  hmac-sha256
Transfer Key Value: [base64 from ioc2rpz GUI → Configuration → TSIG Keys]
Allow Notify:       192.168.122.23
Allow Query:        loopback_only
```

See [Decision: BIND as TSIG intermediary](../decisions/bind-tsig-intermediary.md) for why BIND is needed (Unbound 1.24.2 lacks TSIG support).

**Verify zone transfer:**

```bash
pluginctl bind status
cat /var/log/named/named.log | tail -20
```

Expected:
```
Transfer completed: 97 messages, 71767 records, 1557644 bytes, 0.645 secs
```

**TSIG troubleshooting** — if `tsig indicates error` in named.log:
1. Check clock skew: `date -u` on both pop01 and mgmt01 (max 300s tolerance)
2. Verify key name matches exactly between ioc2rpz and BIND configs
3. In ioc2rpz GUI: ensure the TSIG key is linked to the zone (not just created)

---

## Step 6: Activate Unbound RPZ on pop01

> **Gotcha: Config path is `/usr/local/etc/unbound.opnsense.d/rpz.conf`, NOT `/var/unbound/etc/rpz.conf`.** The `/var/unbound/` chroot is regenerated on every Unbound restart — files placed there are deleted.
> See [Finding: Unbound config path](../findings/unbound-config-path.md).

```bash
vi /usr/local/etc/unbound.opnsense.d/rpz.conf
```

Content:

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

> **Gotcha: `python` MUST be in the module-config.** OPNsense's built-in DNS blocklist runs as a Python script in Unbound. Without `python`, OPNsense DNSBL breaks.

> **Gotcha: `configctl unbound check` gives a false positive.** `unbound-checkconf` has a hardcoded whitelist of module combinations; `respip + python` is not in it but works at runtime. Ignore the error and restart directly:

```bash
configctl unbound restart
```

**Verify module loading:**

```bash
cat /var/log/system.log | grep "init module"
# Must show all 4 modules:
# init module 0: respip
# init module 1: python
# init module 2: validator
# init module 3: iterator
```

The OPNsense WebUI shows "The configuration contains manual overwrites" — this is expected and correct.

---

## Step 7: Configure NetBird DNS primary nameserver

This step is a **prerequisite for BYOD RPZ protection**. Without it, external queries from mobile01 bypass Unbound entirely.

NetBird Dashboard → DNS → Nameservers → Add:

```
IP:             100.70.154.79 (pop01 overlay IP)
Match domains:  (LEAVE EMPTY)
```

> **Gotcha: Empty match-domains = primary nameserver for ALL queries.** Not just `*.sandbox.local`. Without this, queries for external domains go via the adapter's original DNS, and RPZ doesn't protect against external malicious domains.
> See [Finding: NetBird primary nameserver](../findings/netbird-primary-nameserver.md).

---

## Final verification

**Test domain:** `testentry.rpz.urlhaus.abuse.ch` — a permanent test entry in the URLhaus RPZ feed.

**Distinguishing RPZ from regular NXDOMAIN:** An RPZ block has the `aa` (authoritative answer) flag. Regular internet NXDOMAIN does not.

**Test point 1 — pop01 local:**

```bash
drill @127.0.0.1 testentry.rpz.urlhaus.abuse.ch
# Expected: rcode: NXDOMAIN, flags: aa
```

**Test point 2 — mobile01 via NetBird:**

```powershell
nslookup testentry.rpz.urlhaus.abuse.ch
# Expected: Non-existent domain
```

RPZ log should show source IP `100.70.95.98` (mobile01).

**Test point 3 — dc01 via DC-LAN:**

```bash
drill @10.0.0.1 testentry.rpz.urlhaus.abuse.ch
# Expected: rcode: NXDOMAIN, flags: aa
```

**Control test — normal domain works:**

```bash
drill @127.0.0.1 google.com
# Expected: rcode: NOERROR, no aa flag, valid A record
```

---

## Known open points

- **ioc2rpz GUI JS fix is non-persistent** — re-apply after container rebuilds
- **DNS NOTIFY doesn't reach BIND directly** — ioc2rpz sends NOTIFY to port 53, BIND listens on 53530. Feeds update on the 3600s poll interval. Manual trigger: `rndc -p 953 retransfer threat-intel.rpz.sase`

---

## Checklist

- [ ] ioc2rpz containers running on mgmt01
- [ ] GUI accessible and login works
- [ ] 71,767+ RPZ records loaded
- [ ] BIND zone transfer completed successfully
- [ ] Unbound RPZ active (4 modules loaded)
- [ ] NetBird primary nameserver configured (empty match-domains)
- [ ] pop01 local: `testentry.rpz.urlhaus.abuse.ch` → NXDOMAIN + aa
- [ ] mobile01: same test → NXDOMAIN via NetBird DNS relay
- [ ] dc01: same test → NXDOMAIN via DC-LAN
- [ ] Normal domains still resolve

---

## Related

- [Component: ioc2rpz](../components/ioc2rpz.md)
- [Concept: RPZ](../concepts/rpz.md)
- [Decision: ioc2rpz vs Unbound native](../decisions/ioc2rpz-vs-unbound-native.md)
- [Decision: BIND as TSIG intermediary](../decisions/bind-tsig-intermediary.md)
- [Finding: Unbound no TSIG](../findings/unbound-no-tsig.md)
- [Finding: Unbound config path](../findings/unbound-config-path.md)
- [Finding: ioc2rpz GUI JS bug](../findings/ioc2rpz-gui-js-bug.md)
- [Finding: iptables FORWARD ordering](../findings/iptables-forward-ordering.md)
- [Finding: NetBird primary nameserver](../findings/netbird-primary-nameserver.md)

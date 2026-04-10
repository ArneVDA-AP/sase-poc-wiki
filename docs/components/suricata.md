---
title: "Suricata IDS — WAN + LAN Network Inspection"
tags: [suricata, opnsense, ids, ips, network, firewall, sase]
---

# Suricata IDS — WAN + LAN Network Inspection

**Role:** Parallel network-layer inspection on pop01 — detects threats in raw packet flows on WAN (vtnet0) and LAN (vtnet1) using signature-based rules. Complements the ICAP pipeline, which inspects HTTP content; Suricata inspects network flows that bypass the proxy entirely.  
**Version:** Suricata 7.x (OPNsense 25.1 bundled)  
**Config location:** `/usr/local/opnsense/service/templates/OPNsense/IDS/custom.yaml` (persistent config), `/var/log/suricata/suricata_YYYYMMDD.log`, `/var/log/suricata/eve.json`

---

## How it works in this stack

Suricata runs in PCAP capture mode on pop01, reading raw frames from BPF devices on vtnet0 (WAN) and vtnet1 (LAN). It reassembles TCP streams, parses protocol fields, and matches against 79 620+ rules from four rulesets (ET Open, Abuse.ch URLhaus, SSL Fingerprint Blacklist, SSL IP Blacklist).

**What vtnet0 (WAN) sees:** Squid's re-encrypted upstream HTTPS connections, DNS queries from pop01's resolver, other direct TCP/UDP traffic that bypasses the proxy. Suricata extracts TLS metadata (SNI, JA3/JA4 fingerprints), HTTP headers in plaintext (from non-HTTPS flows), DNS query/response data, and matches against C2/botnet IP and domain signatures.

**What vtnet1 (LAN) sees:** All internal DC-LAN traffic from dc01 — unencrypted, all protocols. This catches threats on the internal segment (lateral movement, reconnaissance, software update anomalies from dc01).

**Why not wt0 (NetBird):** WireGuard is a Layer 3 VPN. Traffic routed via wt0 does not appear as ingress frames on that interface from BPF's perspective. `tcpdump` on wt0 with a TCP filter shows 0 packets. Suricata on wt0 would see nothing. See [Finding: wt0 pf rdr limitation](../findings/wt0-pf-rdr-limitation.md) and [Decision: Suricata WAN+LAN](../decisions/suricata-wan-lan.md).

**IDS mode, not IPS:** Suricata runs in IDS mode (PCAP, no packet dropping). Netmap IPS mode requires NIC drivers with native Netmap support (Intel igb/ixgbe, Broadcom bge) — virtio NICs in QEMU have none. Divert IPS requires explicit `pf divert-to` firewall rules that were not configured. The drop/alert policy table is configured and ready for IPS activation on physical hardware. See [Decision: IDS vs IPS](../decisions/ids-vs-ips.md) and [Finding: Suricata Netmap/virtio](../findings/suricata-netmap-virtio.md).

**RAM requirement:** Running Suricata + ClamAV + Squid concurrently requires **minimum 8 GB RAM** on pop01. Hyperscan pattern compilation of ET Open rules peaks at ~4 GB RSS. At 6 GB, the OOM killer silently terminates services without log output.

---

## Configuration

### GUI settings

Services → Intrusion Detection → Administration → Settings:

```
Enabled:          ✔
IPS mode:         ☐  (IDS)
Promiscuous mode: ✔
Pattern matcher:  Hyperscan
Interfaces:       WAN, LAN
Home networks:    10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 100.64.0.0/10
```

`100.64.0.0/10` in HOME_NET is critical: NetBird client IPs fall in the CGNAT range. Without this entry, Suricata treats NetBird peers as external hosts and rules using `$HOME_NET` fire incorrectly.

**Hyperscan selection:** Verify SSE4.2 is available before selecting Hyperscan. Use `dmesg | grep -i features` and look for `SSE4.2` in Features2. `sysctl hw.model` on QEMU VMs shows a generic string and is not reliable.

### Rulesets

Download via Administration → Download. After download, rules are **not automatically active** — go to Administration → Rules and enable all rule files separately.

| Ruleset | Rules |
|---------|-------|
| ET Open (all emerging-* categories) | ~79 620 |
| Abuse.ch URLhaus | malware distribution |
| Abuse.ch SSL Fingerprint Blacklist | JA3 fingerprints |
| Abuse.ch SSL IP Blacklist | known C2 IPs |

### Differentiated drop/alert policy

Via Administration → Policies. Configured and ready for IPS activation:

| Action | Categories |
|--------|-----------|
| **Drop** | emerging-malware, emerging-exploit, emerging-worm, emerging-botcc, emerging-botcc.portgrouped, compromised, drop (Spamhaus), ciarmy, Threatview Cobalt Strike C2, all Abuse.ch feeds |
| **Alert** | tor, emerging-info, emerging-policy, emerging-dns, emerging-web_client, emerging-web_server, emerging-misc, emerging-games, emerging-chat, emerging-p2p, emerging-attack_response |

Drop categories are those where a match is **always** malicious — no legitimate traffic possible. Alert categories carry false positive risk.

### custom.yaml (persistent configuration)

OPNsense regenerates `suricata.yaml` on every GUI Apply. Persistent changes go in:

```
/usr/local/opnsense/service/templates/OPNsense/IDS/custom.yaml
```

**Final working content (post vtnet1 fix):**

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

Apply:
```bash
configctl template reload OPNsense/IDS
configctl ids restart
```

---

## Validation results

Four detection categories confirmed after vtnet1 fix:

| # | Category | Test | SID | Interface |
|---|----------|------|-----|-----------|
| 1 | Attack response | `curl.exe -x http://100.70.154.79:3128 http://testmyids.com/` | 2100498 | vtnet0 |
| 2 | DNS anomaly | Automatic — DNS .biz TLD queries from pop01 resolver | 2027863 | vtnet0 |
| 3 | Suspicious user-agent | `curl.exe -x ... -A "BlackSun" http://example.com` | 2008983 | vtnet0 |
| 4 | Software detection (LAN) | dc01 `apt update` activity | 2013504 | vtnet1 |

---

## Integration points

| Component | Direction | What |
|-----------|-----------|------|
| [Squid](squid.md) | parallel | Suricata on vtnet0 sees Squid's upstream connections — re-encrypted HTTPS; TLS metadata visible |
| [NetBird](netbird.md) | HOME_NET dependency | `100.64.0.0/10` must be in HOME_NET for correct rule evaluation |
| [ioc2rpz/RPZ](ioc2rpz.md) | complementary | Abuse.ch URLhaus in Suricata signatures and RPZ feeds overlap in source; Suricata detects connections, RPZ prevents DNS resolution |

---

## Known issues / gotchas

**vtnet1 `interface: default` generates zero events** — using `interface: default` in custom.yaml resolves to only the primary capture interface (vtnet0), not all interfaces. vtnet1 had a BPF device open but received no packets. Fix: explicit per-interface declarations. See [Finding: Suricata interface default bug](../findings/suricata-interface-default-bug.md).

**`sockstat` does not work for Suricata verification** — Suricata uses BPF, not TCP/UDP sockets. `sockstat -4 | grep suricata` returns nothing even when Suricata is running correctly. Use `procstat -f <PID> | grep bpf` instead.

**Logfile is date-stamped** — `/var/log/suricata/suricata.log` does not exist. The correct path is `/var/log/suricata/suricata_YYYYMMDD.log`.

**Squid connection pooling explains single alerts per test** — when the same testmyids.com URL is requested multiple times through Squid, Suricata sees only one flow (Squid reuses the upstream TCP connection). This generates one alert per SID per flow — correct behavior, not a threshold problem.

**Netmap IPS fails on virtio NICs** — see [Finding: Suricata Netmap/virtio](../findings/suricata-netmap-virtio.md). Switching to Netmap causes zero packets processed and reverts all previous alert activity.

---

## Related

- [Architecture overview](../overview/architecture.md)
- [Concept: Zero Trust](../concepts/zero-trust.md)
- [Decision: Suricata WAN+LAN](../decisions/suricata-wan-lan.md)
- [Decision: IDS vs IPS](../decisions/ids-vs-ips.md)
- [Finding: Suricata interface default bug](../findings/suricata-interface-default-bug.md)
- [Finding: Suricata Netmap/virtio](../findings/suricata-netmap-virtio.md)
- [Finding: wt0 pf rdr limitation](../findings/wt0-pf-rdr-limitation.md)

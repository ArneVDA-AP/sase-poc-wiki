---
title: "Runbook: IDS"
tags: [runbook, suricata, ids, hyperscan]
---

# Runbook: IDS

**Source:** `raw/Doc3_Suricata_IDS.md`
**Node(s):** pop01 (OPNsense — vtnet0 WAN + vtnet1 LAN)
**Prerequisites:** [Runbook 03: Proxy & WPAD](03-proxy-wpad.md) completed (Squid operational)
**Status:** Operational (IDS mode — IPS ready for physical hardware)

---

## Prerequisites checklist

- [ ] pop01 has **8 GB RAM minimum** (see Step 0)
- [ ] OPNsense 25.1 on pop01
- [ ] vtnet0 (WAN) and vtnet1 (LAN) interfaces configured
- [ ] Squid and ClamAV already running

---

## Step 0: Verify RAM — 8 GB minimum

> **Gotcha: This must happen before anything else.** At 4 GB (or even 6 GB), ClamAV and Suricata silently crash via FreeBSD's OOM-killer — no log entries are written. ClamAV uses ~1.2 GB, Suricata ~760 MB after Hyperscan compilation (peaks at ~4 GB during compilation), Squid ~400 MB. Combined they exceed 6 GB.

If pop01 has less than 8 GB: shut down pop01 cleanly (`shutdown -h now`), change RAM in GNS3 node settings, restart.

**Verify after boot:**

```bash
top
# ClamAV: ~1230 MB RSS, Suricata: ~761 MB RSS
# Free: ~3974 MB
```

---

## Step 1: Verify SSE4.2 CPU support

Hyperscan requires SSE4.2. Do not use `sysctl hw.model` — it shows "QEMU Virtual CPU" which is uninformative.

```bash
dmesg | grep -i features
# Look for: SSE4.2 in the Features2 line
```

SSE4.2 present → Hyperscan is available. The Proxmox host passes CPU instructions through because of the `type=host` CPU setting (configured in Runbook 01).

---

## Step 2: Configure Suricata via GUI

Services → Intrusion Detection → Administration → Settings:

```
Enabled:          checked
IPS mode:         unchecked  (IDS — see note below)
Promiscuous mode: checked
Pattern matcher:  Hyperscan
Interfaces:       WAN, LAN
Home networks:    10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 100.64.0.0/10
```

> **Gotcha: `100.64.0.0/10` MUST be in HOME_NET.** NetBird peer IPs use the CGNAT range (100.64.x.x). Without this subnet, Suricata treats NetBird clients as external hosts and rules using `$HOME_NET` won't trigger correctly.

> **Gotcha: Promiscuous mode is essential on vtnet1 (LAN).** Without it, the NIC only sees frames addressed to pop01's own MAC. Traffic from dc01 destined elsewhere would be invisible.

**Why IDS, not IPS:** Netmap IPS requires NIC drivers with native Netmap support (Intel `igb`, `ixgbe`). The virtio NICs in QEMU/GNS3 have no Netmap driver — all worker threads process zero packets. The drop/alert policies are configured and ready for IPS activation on physical hardware with compatible NICs. See [Decision: IDS vs IPS](../decisions/ids-vs-ips.md).

---

## Step 3: Download and enable rulesets

Via Administration → Download, select and click "Download & Update Rules":

| Ruleset | Content |
|---------|---------|
| ET Open (all emerging-* categories) | ~79,620 rules |
| Abuse.ch URLhaus | Malware distribution domains/IPs |
| Abuse.ch SSL Fingerprint Blacklist | JA3 fingerprints of known malware TLS |
| Abuse.ch SSL IP Blacklist | Known malware C2 server IPs |

> **Gotcha: Downloaded rules are NOT automatically activated.** Go to Administration → Rules, select all rule files, and explicitly **Enable** each one. This is a separate step.

---

## Step 4: Create differentiated policies

Via Administration → Policies:

**Policy 1 — Drop (high-confidence, always malicious):**
- Categories: emerging-malware, emerging-exploit, emerging-worm, emerging-botcc, emerging-botcc.portgrouped, compromised, drop, ciarmy, Threatview_CS-C2, all Abuse.ch feeds
- Action: Drop, New Action: Drop

**Policy 2 — Alert (false positive risk):**
- Categories: tor, emerging-info, emerging-policy, emerging-dns, emerging-web_client, emerging-web_server, emerging-misc, emerging-games, emerging-chat, emerging-p2p, emerging-attack_response
- Action: Alert, New Action: Alert

Note: `emerging-attack_response` is set to Alert — this enables test validation with SID 2100498.

---

## Step 5: Create custom.yaml for explicit per-interface PCAP

OPNsense regenerates `/var/unbound/suricata.yaml` on every GUI Apply. Use the persistent custom config path instead:

```bash
vi /usr/local/opnsense/service/templates/OPNsense/IDS/custom.yaml
```

> **Gotcha: `interface: default` resolves to vtnet0 ONLY.** This was the root cause of a full troubleshooting session — vtnet1 had a BPF device open but Suricata never sent packets to it. Result: 0 events from vtnet1 despite confirmed BPF capture. Each interface needs an explicit declaration.
> See [Finding: Suricata interface default bug](../findings/suricata-interface-default-bug.md).

Use this exact content — the final working version after the vtnet1 fix:

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

## Step 6: Verify Suricata is capturing

> **Gotcha: `sockstat` does NOT work for Suricata.** Suricata uses BPF for raw packet capture, not TCP/UDP sockets. `sockstat -4 | grep suricata` returns empty output even when Suricata is running correctly.

**Correct verification:**

```bash
# Get PID:
ps aux | grep suricata

# Verify BPF capture:
procstat -f <PID> | grep bpf
# Must show bpf0, bpf1 — one per configured interface
```

> **Gotcha: Log file is date-stamped.** The path is `/var/log/suricata/suricata_YYYYMMDD.log`, not `suricata.log`.

```bash
cat /var/log/suricata/suricata_$(date +%Y%m%d).log | head -30
```

**RAM peak during Hyperscan compilation:** CPU spikes to 143%+ for 1-2 minutes on startup. This is normal — wait for compilation to finish.

---

## Final verification

Run these four tests to prove four different detection categories:

| # | Test | Command | Expected SID | Interface |
|---|------|---------|-------------|-----------|
| 1 | Attack response | `curl.exe -x http://100.70.154.79:3128 http://testmyids.com/` from mobile01 | 2100498 | vtnet0 (WAN) |
| 2 | DNS anomalies | Automatic — DNS .biz TLD queries from pop01 | 2027863 | vtnet0 (WAN) |
| 3 | Suspicious User-Agent | `curl.exe -x http://100.70.154.79:3128 -A "BlackSun" http://example.com` | 2008983 | vtnet0 (WAN) |
| 4 | LAN traffic | `apt update` on dc01 | 2013504 | vtnet1 (LAN) |

Verify vtnet1 events exist:

```bash
grep '"in_iface":"vtnet1"' /var/log/suricata/eve.json | wc -l
# Must be > 0
```

Check OPNsense WebUI → Services → Intrusion Detection → Alerts: both WAN and LAN alerts should be visible.

**Note on repeated tests:** Multiple curl requests to the same domain may produce only one alert. This is not a threshold bug — Squid reuses upstream TCP connections (connection pooling), so Suricata sees one flow and correctly generates one alert per SID per flow.

---

## Checklist

- [ ] pop01 has 8 GB RAM, all services stable
- [ ] SSE4.2 confirmed, Hyperscan selected
- [ ] `100.64.0.0/10` in HOME_NET
- [ ] 79,620+ rules downloaded AND enabled
- [ ] Drop/Alert policies configured
- [ ] `procstat -f <PID> | grep bpf` shows bpf0 and bpf1
- [ ] vtnet0 alerts present (SID 2100498 or similar)
- [ ] vtnet1 alerts present (dc01 traffic)

---

## Related

- [Component: Suricata](../components/suricata.md)
- [Decision: IDS vs IPS](../decisions/ids-vs-ips.md)
- [Decision: Suricata WAN+LAN](../decisions/suricata-wan-lan.md)
- [Finding: Suricata interface default bug](../findings/suricata-interface-default-bug.md)
- [Finding: Suricata Netmap/virtio](../findings/suricata-netmap-virtio.md)

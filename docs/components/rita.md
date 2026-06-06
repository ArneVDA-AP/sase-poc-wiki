---
title: "RITA — Behavioral Threat Analytics"
tags: [rita, zeek, behavioral-analysis, beaconing, c2, dns, rpz, docker, sase]
---

# RITA — Behavioral Threat Analytics

**Role:** Analyses [Zeek](zeek.md) logs for behavioral indicators a signature engine cannot see — beaconing, C2-over-DNS, long connections, threat-intel matches — and surfaces suspicious domains that can be pushed into the [RITA→RPZ pipeline](../decisions/rita-rpz-automation.md) for DNS-level blocking.  
**Version:** RITA v5.1.1 (`ghcr.io/activecm/rita`), ClickHouse 24.1.6 backend  
**Status:** ✅ PoC-validated on the parallel stack — sandbox integration pending  
**Config location:** `/etc/rita/config.hjson` (scoring, filtering, threat-intel), `/opt/rita/.env`, `/opt/rita/docker-compose.yml` on mgmt01 (`192.168.122.20`)

> **Sidenote, parallel-stack scope.** RITA runs on the **SASE_POC** topology built by Rayan, on mgmt01 (`192.168.122.20`). It is validated end-to-end there but **not integrated into the main sandbox** control plane. Its findings reach the sandbox only through the RITA→RPZ feed file (see Integration points), not via NATS or the [Control Daemon](control-daemon.md). RITA is the behavioral counterpart to the sandbox's signature-based [Suricata IDS](suricata.md); where the two overlap, the sandbox is the source of truth.

---

## How it works in this stack

RITA (Real Intelligence Threat Analytics) reads the logs [Zeek](zeek.md) produces and looks for *patterns over time* rather than known-bad content in a single packet. Zeek sees, RITA decides. It imports Zeek's `conn`, `dns`, `ssl`, `http` and `ntp` logs into a **ClickHouse** column store (which replaced MongoDB from RITA v4) and runs its analytics there.

The core question RITA answers is "does this internal host talk to that destination in a way that looks automated rather than human?" A user browsing is irregular; malware checking in with its C2 is metronomic. RITA scores each `source → destination(FQDN/SNI)` pair on how regular the interval and payload size are, producing a **beacon score** from 0 to 1.

### What RITA detects

- **Beaconing** — periodic, regular call-home behavior. The primary use case here.
- **C2-over-DNS** — command-and-control tunnelled through DNS queries (subdomain enumeration patterns), which the [DNS-level RPZ blocking](../concepts/rpz.md) cannot catch on its own because the queries themselves carry the payload.
- **Long connections** — sessions held open far longer than normal, a classic C2 tell.
- **Threat-intel matches** — destinations appearing on a feed (Feodo tracker) loaded into RITA.

### Import model

Zeek rotates logs hourly into `/opt/zeek/logs/YYYY-MM-DD/`. RITA imports those into a named database. The operational dataset is a **rolling** import (`--rolling`), which enables first-seen scoring, prevalence tracking and temporal analysis across the whole window rather than a single snapshot:

```bash
sudo rita import -l /opt/zeek/logs/ --database sase_poc --rolling
```

The initial 12-day import (April 7–21, 2026) processed in roughly 7.5 minutes. Database names cannot contain hyphens in RITA v5 — use `sase_poc`, not `sase-poc`.

### Reading the analysis

```bash
sudo rita view sase_poc --beacon           # beaconing (primary)
sudo rita view sase_poc --long-connection  # long-held sessions
sudo rita view sase_poc --c2-over-dns      # DNS-tunnelled C2
sudo rita view sase_poc --threat-intel     # feed matches
```

### Validation — legitimate beacons prove the pipeline

The first validation run flagged only legitimate periodic services, which is exactly the proof that the detection works: anything regular shows up, and the analyst investigates whatever they do not recognise.

| Source | Destination | Beacon score | Interpretation |
|--------|-------------|--------------|----------------|
| `192.168.122.20` | pkgs.netbird.io | 100% | NetBird update check — perfectly regular |
| `192.168.122.20` | 185.125.190.56 | 99% | Ubuntu/Canonical NTP sync |
| `192.168.122.20` | 8.8.8.8 | 73% | Google DNS, periodic |

In a compromised environment a malicious C2 beacon would appear in this same list next to the benign entries. That those entries land near 100% on a known-good network confirms the scoring is sound.

### The `grafana.com` example — detection to block

The end-to-end value of RITA is realised when a detection feeds the [RITA→RPZ pipeline](../decisions/rita-rpz-automation.md). In the May threat-intel session, RITA scored `grafana.com` as **Critical (beacon score ≈ 0.998)** from a simulated periodic connection. An extraction script lifted that FQDN from the `rita view --beacon` output into a feed file, ioc2rpz aggregated it into the `threat-intel.rpz.sase` zone, and Unbound began answering **NXDOMAIN** for it. The behavioral detection on one stack became a DNS block enforced on clients:

```
nslookup grafana.com → Non-existent domain   (RITA-detected beacon, score 0.998)
```

This is the pipeline RITA exists to feed. See the integration note below and [Decision: RITA→RPZ automation](../decisions/rita-rpz-automation.md).

---

## Configuration

The backend is three Docker containers (RITA, ClickHouse, syslog-ng) started with `docker compose up -d` from `/opt/rita`, persistent via `restart: unless-stopped`. Wait for the ClickHouse healthcheck before importing:

```bash
docker inspect rita-clickhouse --format='{{.State.Health.Status}}'   # expect: healthy
```

Analysis behaviour lives in `/etc/rita/config.hjson`: beacon scoring thresholds, connection filtering, and the threat-intel feed list. Two databases exist from validation: `sase_poc` (rolling, primary) and `sase_test` (single-day static, used during bring-up).

**Score, not severity, is the reliable signal for automation.** RITA's own severity label is conservative — a destination with hundreds of connections scored only "Medium" in testing. The RITA→RPZ extraction therefore filters on **beacon score ≥ 0.8**, not on the severity field. See [Decision: RITA→RPZ automation](../decisions/rita-rpz-automation.md).

---

## Integration points

| Component | Direction | What |
|-----------|-----------|------|
| [Zeek](zeek.md) | ← (consumes) | RITA imports Zeek's `conn/dns/ssl/http/ntp` logs from `/opt/zeek/logs/` into ClickHouse. No Zeek, no RITA input. |
| [ioc2rpz / RPZ](ioc2rpz.md) | → (feed file) | A cron extraction script writes high-score FQDNs to a `rita_beacons.txt` feed that ioc2rpz reads as a source, turning a behavioral detection into a DNS NXDOMAIN. This is the only path RITA findings reach the sandbox today. See [Decision: RITA→RPZ automation](../decisions/rita-rpz-automation.md) and [Runbook: RITA→RPZ integration](../runbooks/15-rita-rpz-integration.md). |
| [Suricata](suricata.md) | complementary | Suricata answers "is this packet known-bad?"; RITA answers "is this flow behaving like C2?". Behavioral vs signature, on the same traffic. |

---

## Known issues / gotchas

**Rolling import evaluates the window, but only ingests the latest chunk.** A `rita import --rolling` for the current day processes the most recent hour-chunk; a domain that only beaconed in an earlier chunk scores low until that chunk is in the window. Import several hour-chunks for a complete picture, or run a beacon test long enough to fall inside the active analysis window.

**CDN rotation dilutes beacon scores.** A destination behind a CDN (rotating answer IPs) splits across many `src→dst` pairs and scores lower than its true regularity — one test destination with 333 connections landed at only 0.629. Forcing a stable IP (`curl --resolve`) restores a clean high-score beacon.

**Whitelist legitimate periodic services before automating.** NetBird update checks (100%) and Ubuntu NTP (99%) are textbook beacons. Feeding RITA output into RPZ without a whitelist would NXDOMAIN your own update and time infrastructure. The whitelist is the first false-positive control in the [RITA→RPZ pipeline](../decisions/rita-rpz-automation.md).

**Not on the sandbox event bus.** RITA publishes nothing to NATS and does not trigger the [Control Daemon](control-daemon.md). Its only sandbox-facing output is the RPZ feed file; everything else stays in the CLI views on the parallel stack.

---

## Related

- [Architecture overview](../overview/architecture.md)
- [Component: Zeek](zeek.md)
- [Component: ioc2rpz + RPZ](ioc2rpz.md)
- [Component: Suricata](suricata.md)
- [Concept: Behavioral analysis](../concepts/behavioral-analysis.md)
- [Concept: RPZ](../concepts/rpz.md)
- [Decision: RITA→RPZ automation](../decisions/rita-rpz-automation.md)
- [Finding: DC segment mirror limit](../findings/dc-segment-mirror-limit.md)
- [Runbook: Zeek & RITA](../runbooks/14-zeek-rita.md)
- [Runbook: RITA→RPZ integration](../runbooks/15-rita-rpz-integration.md)

---
title: "Telemetry Stack (Grafana + Prometheus + Loki)"
tags: [grafana, prometheus, loki, telemetry, docker, nats-jetstream, netbird, wazuh, siem]
---

# Telemetry Stack (Grafana + Prometheus + Loki)

**Role:** Parallel-stack observability layer — aggregates metrics (Prometheus), logs (Loki), and SASE security telemetry into a single Grafana "single pane of glass" for the PoC environment.  
**Status:** ✅ PoC-validated on the parallel stack — sandbox integration pending  
**Version:** Grafana 11.6.1, Prometheus (latest, pinned in compose), Loki 3.4.3  
**Config location:** mgmt01, `/opt/sase-monitor/` (Docker Compose)

> **Sidenote — scope.** This stack is built and validated on the parallel SASE_POC stack (mgmt01), but is **not yet integrated into the main sandbox**. The sandbox's own observability is **Wazuh SIEM** (see [Wazuh](wazuh.md)), which remains the source of truth for security alerting. The telemetry stack is a complementary monitoring layer, not a replacement for Wazuh.

---

## How it works in this stack

The telemetry stack answers the SASE-observability question: who has access (NetBird peer events), what are they doing (Zeek conn/dns/http), are threats detected (Wazuh/Suricata), and is the infrastructure healthy (GNS3 node status + node_exporter). Rather than one tool per device, everything surfaces through one Grafana interface.

Three storage backends sit behind Grafana:

- **Prometheus** — pull-based metrics database. Runs in `network_mode: host` so it can scrape exporters that live both inside Docker (GNS3 exporter) and outside it (node_exporter on mgmt01 and dc01).
- **Loki** — lightweight log store (a leaner alternative to Elasticsearch), 7-day retention.
- **Grafana** — visualisation and dashboards, provisioned from files so dashboards survive container rebuilds.

Logs do not reach Loki via Promtail in the current build. Instead, a set of small Python forwarder containers feed Loki, and most log sources flow through the existing **NATS JetStream** bus (built by a teammate) rather than through a dedicated syslog receiver. See [Architecture: two builds](#architecture-two-builds-poc-1a-vs-mgmt01) for why.

### Containers (mgmt01 build)

| Container | Image | Network | Function |
|-----------|-------|---------|----------|
| `mon-grafana` | grafana/grafana:11.6.1 | monitor-net | Dashboard UI (`:3000`) |
| `mon-prometheus` | prom/prometheus | host | Metrics scraping |
| `mon-loki` | grafana/loki:3.4.3 | monitor-net | Log storage (`:3100`) |
| `mon-nats-loki-forwarder` | custom Python | monitor-net + NATS net | Subscribes to `logs.>` on NATS, pushes to Loki |
| `mon-gns3-exporter` | python:3.12-slim | host | GNS3 API → Prometheus (`:9456`) |
| `mon-zeek-nats-publisher` | custom Python | monitor-net + NATS net | Tails Zeek spool → NATS |
| `mon-wazuh-loki-forwarder` | custom Python | host | Wazuh OpenSearch (`level >= 5`) → Loki |
| `mon-netbird-api-poller` | custom Python | monitor-net + netbird net | NetBird REST API → Loki |

The forwarder design is deliberately container-per-source so that one broken feed cannot stall the others.

---

## Dashboards

Four dashboards live in the Grafana "SASE PoC" folder, provisioned from JSON files (`/opt/sase-monitor/grafana/dashboards/`). Provisioned dashboards and datasources are **read-only in the UI** — changes must be made in the YAML/JSON files plus a container restart.

| Dashboard | Datasource | Shows |
|-----------|------------|-------|
| **Node Status** (GNS3) | Prometheus | One stat panel per GNS3 node, GREEN = started / RED = stopped; counters for nodes started/stopped. Query: `gns3_node_status` |
| **VM Metrics** (Host Metrics) | Prometheus | CPU / RAM / disk I/O / network for mgmt01 (`node_exporter`). dc01 panels present but empty (target DOWN, see gotchas) |
| **OPNsense Log Stream** (SASE Security Logs) | Loki | Live log stream with Device + Component dropdown filters; firewall events; log volume per device. Query: `{job=~"sase-logs\|wazuh-alerts", device=~"$device", category=~"$category"}` |
| **Stack Health** | Prometheus | Prometheus target UP/DOWN, Loki ingestion rate, scrape duration. Query: `up{job='...'}` |

NetBird ZTNA data appears in the Security Logs dashboard under `category=netbird_api`; Wazuh alerts under `device=wazuh`. A dedicated NetBird/ZTNA dashboard is a planned extension, not yet built.

---

## Log pipeline

In the mgmt01 build, 17 log categories are active in Loki across three devices (pop01, mgmt01, wazuh):

```
caddy, clamav, firewall, m365, netbird, netbird_api, system, unbound,
wazuh, zeek_conn, zeek_dns, zeek_http, zeek_notice, zeek_ssh, zeek_ssl,
zeek_syslog, zeek_weird
```

The forwarders derive Loki labels from the NATS subject: `logs.pop01.firewall` becomes `device=pop01, category=firewall`. A key design choice is using `time.time()` (Loki ingest time) as the log timestamp rather than the original log timestamp — OPNsense (FreeBSD) emits timestamps that claim `+00:00` while actually being CEST, which poisons Loki streams with "entry too far behind" rejections. See [Finding: RFC3164 vs RFC5424 syslog](../findings/rfc3164-vs-rfc5424-syslog.md) for the related syslog-format problem in the earlier build.

The **NetBird API poller** is worth calling out: the NetBird management container logs almost nothing useful (~10 lines/hour of noise), so the rich ZTNA data — which peers are connected, `last_seen`, policy changes, who did what — is pulled from the NetBird REST API every 60 seconds and written to Loki as `category=netbird_api`. This contrasts with [NetBird](netbird.md) container logs (`category=netbird`), which are kept but are structurally too thin for observability.

---

## Integration points

| Interface | Direction | Details |
|-----------|-----------|---------|
| NATS JetStream (`logs.>`) | Inbound | `mon-nats-loki-forwarder` is a durable consumer; carries OPNsense syslog, Zeek, Caddy, NetBird, M365 into Loki |
| [GNS3](gns3.md) REST API | Inbound (poll) | `mon-gns3-exporter` polls `192.168.122.1:3080` every 15s → Prometheus metric `gns3_node_status` |
| node_exporter (`:9100`) | Inbound (scrape) | mgmt01 (UP) and dc01 (DOWN — OPNsense blocks the scrape) |
| [Wazuh](wazuh.md) OpenSearch (`:9200`) | Inbound (poll) | `mon-wazuh-loki-forwarder` polls `level >= 5` alerts → Loki `category=wazuh` |
| [NetBird](netbird.md) REST API (`/api/peers`, `/api/events`) | Inbound (poll) | `mon-netbird-api-poller` via Caddy HTTPS → Loki `category=netbird_api` |
| [Caddy](caddy.md) | → browser | `grafana.sase.local` reverse-proxied through Caddy (`tls internal`) for NetBird-connected peers |

### Relationship to Wazuh SIEM

Wazuh and this telemetry stack overlap deliberately but serve different roles. Wazuh is the **sandbox source of truth** for security detection and CASB enforcement: it parses, scores, and (optionally) remediates. The telemetry stack **reads from** Wazuh (the `wazuh-loki-forwarder` pulls `level >= 5` alerts into Loki) so that Grafana can show security alerts next to infrastructure metrics in one view. The telemetry stack does not detect or enforce; it aggregates and visualises. See [Wazuh](wazuh.md).

---

## Architecture: two builds (poc-1a vs mgmt01)

There are **two source documents** describing this telemetry work, on **two different hosts**. They do not contradict each other — the second is a migration of the first — but anyone reading the raw sources should know which is current.

| | **Build v1 (earlier)** | **Build v2 (current)** |
|---|---|---|
| Source | `rayan_GNS3_Telemetry_Documentatie.md` (April 2026) | `SASE_Monitoring_Definitief_Verslag.md` (v3.0, May 2026) |
| Host | poc-1a, `10.158.10.67` (Proxmox VM, outside PoC scope) | mgmt01, `192.168.122.20` (inside the PoC) |
| Log pipeline | network devices → **rsyslog** relay → **Promtail** (TCP 1514) → Loki | NATS JetStream → Python forwarders → Loki |
| Syslog ingest | Promtail syslog receiver; rsyslog converts RFC3164→RFC5424 | No Promtail; no port-514 receiver (514 is taken by Wazuh) |
| Reverse proxy | Nginx on `:8080` | Caddy (reused from NetBird) |
| Extras | — | NetBird API poller, Wazuh forwarder, Zeek publisher |

> ⚠️ **Contradiction (host/scope).** The April document fixes the host as **poc-1a (`10.158.10.67`)** and the May document fixes it as **mgmt01 (`192.168.122.20`)**, explicitly stating the stack was *migrated* from poc-1a ("outside PoC scope") to mgmt01 ("inside the PoC"). Per the wiki authority model, the **later verslag (mgmt01) is the current state**. The poc-1a build is retained here only as the earlier evolution. The May document also notes that "the wiki" lists mgmt01 as `.23`, but `.20` is correct for the SASE_POC (team) project; note that the *sandbox* mgmt01 is `192.168.122.23` (see [GNS3](gns3.md)) — these are two different projects on the same GNS3 host.

The April (poc-1a) build used a Promtail syslog receiver that choked on the RFC3164 format network devices send, which is why an rsyslog relay was introduced. The May (mgmt01) build sidesteps this entirely by riding the existing NATS bus. The RFC3164/RFC5424 lesson is documented as a [finding](../findings/rfc3164-vs-rfc5424-syslog.md) because it is a recurring trap in log pipelines, even though the current build no longer hits it.

---

## Known issues / gotchas

- **Loki bridge IP changes after restart (Prometheus target DOWN).** Docker assigns bridge IPs dynamically; after a `mon-loki` restart its IP can shift (e.g. `.3` → `.2`), and the Prometheus `loki` scrape target goes DOWN with `connection refused`. Fix: `docker inspect mon-loki` for the new IP, update `prometheus.yml`, then `docker restart mon-prometheus`. The durable fix is to put Prometheus on `monitor-net` and target `mon-loki` by hostname, or use a static IP.
- **Prometheus on host network cannot resolve container names.** Host-network Prometheus has no Docker DNS, so it must target containers by their bridge gateway IP (`172.x.x.1`), not by name. Grafana (on `monitor-net`) reaches Prometheus via the Docker gateway IP, not `localhost`.
- **dc01 metrics unreachable.** The `node-dc01` Prometheus target (`10.0.0.100:9100`) is DOWN — OPNsense (pop01) blocks TCP from the WAN segment to the DC-LAN `:9100`. A WAN firewall rule was created but does not match correctly. See [Finding: DC-LAN isolation](../findings/dc-lan-isolation-route-acl.md) for the same DC-LAN reachability pattern.
- **Loki "entry too far behind" / stream poisoning.** A prior `TZ=Europe/Brussels` setting on Loki poisoned streams; recovery required wiping the Loki data volume (`docker volume rm sase-monitor_loki-data`), which is destructive. Avoided going forward by using `time.time()` as the ingest timestamp.
- **Double Zeek publication after reboot.** A teammate's `zeek-to-nats.py` cron job and the `mon-zeek-nats-publisher` container use the same NATS credentials on the same subjects; after a reboot both can run and produce duplicate Zeek entries. Check `ps aux | grep zeek-to-nats` and `docker ps | grep zeek-nats` after every reboot.
- **Docker vs UFW.** Docker manipulates iptables directly and bypasses UFW. In the v1 build only ports 8080 (Nginx) and 514/UDP (rsyslog) were exposed; backends stayed on internal Docker networks. Network-level filtering is delegated to the Proxmox firewall.

---

## Related

- [Component: GNS3](gns3.md)
- [Component: Wazuh](wazuh.md)
- [Component: NetBird](netbird.md)
- [Component: Caddy](caddy.md)
- [Decision: Grafana over a custom UI](../decisions/grafana-vs-custom-ui.md)
- [Finding: RFC3164 vs RFC5424 syslog](../findings/rfc3164-vs-rfc5424-syslog.md)
- [Runbook: Telemetry Stack](../runbooks/16-telemetry.md)

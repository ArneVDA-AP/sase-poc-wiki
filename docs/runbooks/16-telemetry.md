---
title: "Runbook: Telemetry Stack"
tags: [runbook, grafana, prometheus, loki, telemetry, docker]
---

# Runbook: Telemetry Stack

**Node(s):** mgmt01 (Docker — Grafana + Prometheus + Loki + forwarders), pop01 + dc01 (node_exporter)
**Prerequisites:** Docker on mgmt01; [Runbook 10: NATS JetStream](10-nats-jetstream.md) operational (log forwarders consume the NATS bus)
**Status:** ✅ PoC-validated on the parallel stack — sandbox integration pending

> **Scope.** This stack runs on the parallel SASE_POC stack (mgmt01) and is **not yet integrated into the main sandbox**. The sandbox's security observability is [Wazuh SIEM](../components/wazuh.md), which stays the source of truth. This runbook stands up the complementary metrics/log visualisation layer.

---

## Prerequisites checklist

- [ ] Docker and Docker Compose installed on mgmt01
- [ ] NATS JetStream operational on mgmt01 ([Runbook 10](10-nats-jetstream.md)) — the `LOGS` stream carries the log feeds
- [ ] Wazuh stack reachable on mgmt01 (OpenSearch `localhost:9200`) for the Wazuh forwarder ([Runbook 11](11-wazuh.md))
- [ ] GNS3 REST API reachable at `192.168.122.1:3080`
- [ ] SSH access to mgmt01 (and dc01, for node_exporter)

---

## Step 1: Lay out the Docker stack on mgmt01

The stack lives in `/opt/sase-monitor/`. Create the directory structure for the three backends, the dashboards, and the forwarder build contexts:

```
/opt/sase-monitor/
├── docker-compose.yml
├── prometheus/prometheus.yml
├── loki/loki-config.yml
├── grafana/provisioning/{datasources,dashboards}/
├── grafana/dashboards/*.json
├── gns3-exporter/exporter.py
├── nats-loki-forwarder/{forwarder.py,Dockerfile}
├── zeek-nats-publisher/{publisher.py,Dockerfile}
├── wazuh-loki-forwarder/{forwarder.py,Dockerfile}
└── netbird-api-poller/{poller.py,Dockerfile,caddy-root.crt}
```

> **Gotcha: write config files with Python, not bash heredocs.** Special characters (`{}`, `${}`, `\n`) get mangled by bash heredocs and corrupt the file. Use `sudo python3 -c "open(...).write(...)"` for every config file.

---

## Step 2: Configure Prometheus (host network)

Prometheus runs in `network_mode: host` so it can scrape exporters both inside and outside Docker.

> **Gotcha: host-network Prometheus has no Docker DNS.** It cannot resolve container names — target containers by their bridge **gateway** IP (`172.x.x.1`), and target host-level exporters by `localhost`.

`prometheus/prometheus.yml` scrape targets:

| Job | Target | Notes |
|-----|--------|-------|
| `prometheus` | `localhost:9090` | self-monitoring |
| `node-mgmt01` | `localhost:9100` | mgmt01 host metrics |
| `loki` | `172.x.0.2:3100` | **this IP drifts after a Loki restart** — see Step 7 |
| `gns3` | `localhost:9456` | GNS3 node status |
| `node-dc01` | `10.0.0.100:9100` | currently DOWN (OPNsense blocks the scrape) |

Set retention: `--storage.tsdb.retention.time=7d --storage.tsdb.retention.size=5GB`.

---

## Step 3: Configure Loki

`loki/loki-config.yml` critical limits:

```yaml
limits_config:
  retention_period: 168h            # 7 days
  reject_old_samples: false
  reject_old_samples_max_age: 720h  # accept up to 30 days old
  creation_grace_period: 3h         # covers the OPNsense CEST/UTC offset
```

> **Gotcha: do not set `TZ=Europe/Brussels` on the Loki container.** It poisons streams with "entry too far behind" and the only recovery is wiping the Loki volume (destructive). The forwarders use `time.time()` (ingest time) as the Loki timestamp specifically to avoid this. See [Component: Telemetry Stack](../components/telemetry-stack.md).

---

## Step 4: Deploy the log forwarders

Logs reach Loki through small Python containers rather than Promtail (port 514 is occupied by Wazuh, and the NATS bus already exists):

1. **`mon-nats-loki-forwarder`** — a durable consumer of `logs.>` on NATS JetStream. Derives Loki labels from the subject (`logs.pop01.firewall` → `device=pop01, category=firewall`). Carries OPNsense syslog, Zeek, Caddy, NetBird container logs, and M365.
2. **`mon-zeek-nats-publisher`** — tails the Zeek spool (`/opt/zeek/spool/manager/*.log`) and publishes to `logs.mgmt01.zeek.{type}`.
3. **`mon-wazuh-loki-forwarder`** — polls Wazuh OpenSearch (`level >= 5`) every 15s and writes to Loki `category=wazuh`. See [Runbook 11: Wazuh](11-wazuh.md).
4. **`mon-gns3-exporter`** — polls the GNS3 API every 15s and exposes `gns3_node_status{project,node,type,status}` (1=started, 0=stopped) on `:9456`.

> **Gotcha: double Zeek publication after reboot.** A teammate's `zeek-to-nats.py` cron job uses the same NATS credentials/subjects as `mon-zeek-nats-publisher`. After a reboot both can run and duplicate entries. Check `ps aux | grep zeek-to-nats` and `docker ps | grep zeek-nats`.

> **Gotcha: Zeek publisher can stick in a warning loop.** If it hits a NATS connection error on first scan it loops on "no active Zeek logs found" even after the files appear. Fix with `docker restart mon-zeek-nats-publisher`.

---

## Step 5: Deploy the NetBird API poller

The NetBird container logs are too thin for ZTNA observability, so a poller pulls the REST API every 60s. See [Component: NetBird](../components/netbird.md).

1. Create a NetBird **Personal Access Token** (PAT) via the dashboard UI only (Team → Users → your own account). PATs start with `nbp_`. A setup key (UUID) is **not** a PAT.
2. The management container binds port 80 on `127.0.0.1` only, so the poller must reach the API **through Caddy over HTTPS**, not directly.
3. Bake the Caddy root CA into the poller image so Python trusts the `tls internal` cert:
   ```dockerfile
   COPY caddy-root.crt /usr/local/share/ca-certificates/caddy-root.crt
   RUN update-ca-certificates
   ```
4. Add `extra_hosts: ["netbird.sase.local:192.168.122.20"]` so the container can resolve the Caddy hostname.

> **Gotcha: API path has no version prefix on v0.66.4.** The docs say `/api/v1/peers`; this version returns 404. Use `/api/peers` and `/api/events`. Always confirm with `curl` first.

Build and start:

```bash
cd /opt/sase-monitor && sudo docker compose up -d --build netbird-api-poller
sleep 30 && sudo docker logs mon-netbird-api-poller --tail 20
```

Output should list peers (`Peer seen: WS-Rayan (connected)`) and events.

---

## Step 6: Install node_exporter on the monitored hosts

On mgmt01 (and dc01) install `node_exporter` v1.9.1 as a systemd service on `:9100`:

```bash
ssh <user>@<host> 'bash -s' << 'SSHEOF'
cd /tmp
wget -q https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz
tar xzf node_exporter-1.9.1.linux-amd64.tar.gz
sudo mv node_exporter-1.9.1.linux-amd64/node_exporter /usr/local/bin/
sudo useradd -rs /bin/false node_exporter 2>/dev/null || true
sudo tee /etc/systemd/system/node_exporter.service << 'EOF'
[Unit]
Description=Node Exporter
After=network.target
[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter --web.listen-address=0.0.0.0:9100
Restart=always
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo ufw allow 9100/tcp
SSHEOF
```

> **Note: dc01 stays DOWN until OPNsense allows the scrape.** dc01 (`10.0.0.100`) sits behind OPNsense on the DC-LAN; pop01 blocks TCP from the WAN segment to `:9100`. Fixing this is a firewall-rule task on pop01 (`pfctl -sr | grep 9100`), tracked separately. See [Finding: DC-LAN isolation](../findings/dc-lan-isolation-route-acl.md).

---

## Step 7: Start the stack and verify

```bash
cd /opt/sase-monitor && sudo docker compose up -d
sleep 15

echo "=== Grafana ==="    && curl -s http://localhost:3000/api/health
echo "=== Prometheus ===" && curl -s http://localhost:9090/-/healthy
echo "=== Loki ==="       && curl -s http://localhost:3100/ready
```

Check all Prometheus targets:

```bash
curl -s http://localhost:9090/api/v1/targets | python3 -c "
import json,sys
for t in json.load(sys.stdin)['data']['activeTargets']:
    print(f'{t[\"health\"]:6s} {t[\"labels\"][\"job\"]:14s} {t.get(\"lastError\",\"\")[:50]}')"
```

> **Gotcha: Loki target DOWN after a restart = IP drift.** Docker reassigns the `mon-loki` bridge IP on restart, so the Prometheus `loki` target reports `connection refused`. Fix:
> ```bash
> sudo docker inspect mon-loki | grep '"IPAddress"'
> sudo python3 -c "open('/opt/sase-monitor/prometheus/prometheus.yml','w').write(open('/opt/sase-monitor/prometheus/prometheus.yml').read().replace('OLD_IP','NEW_IP'))"
> sudo docker restart mon-prometheus
> ```
> Durable fix: put Prometheus on `monitor-net` and target `mon-loki` by hostname.

---

## Step 8: Verify dashboards in Grafana

Grafana provisions four dashboards (folder "SASE PoC") from JSON files; they survive container rebuilds.

| Dashboard | What to confirm |
|-----------|-----------------|
| Node Status | GNS3 nodes show GREEN/RED; `gns3_node_status` returns data |
| VM Metrics | mgmt01 CPU/RAM/disk/network populated (dc01 empty until Step 6 note resolved) |
| OPNsense Log Stream | Live logs flow; Device/Component dropdowns list pop01/mgmt01/wazuh |
| Stack Health | All Prometheus targets UP except `node-dc01` |

Confirm the active Loki categories (expect 17):

```bash
curl -s -u admin:'<password>' \
  "http://localhost:3000/api/datasources/proxy/uid/<LOKI_UID>/loki/api/v1/label/category/values" \
  | python3 -c "import json,sys; print(json.load(sys.stdin).get('data',[]))"
```

> **Note: provisioned datasources/dashboards are read-only in the UI.** Edit the YAML/JSON files and restart the container; UI edits do not persist.

---

## Checklist

- [ ] Grafana, Prometheus, Loki containers healthy on mgmt01
- [ ] Prometheus targets UP: `prometheus`, `node-mgmt01`, `loki`, `gns3` (dc01 DOWN expected)
- [ ] `mon-nats-loki-forwarder` consuming `logs.>` → Loki
- [ ] `mon-zeek-nats-publisher` and `mon-wazuh-loki-forwarder` feeding Loki
- [ ] `mon-gns3-exporter` exposing `gns3_node_status` on `:9456`
- [ ] `mon-netbird-api-poller` writing `category=netbird_api` (PAT valid, Caddy CA trusted)
- [ ] node_exporter active on mgmt01 (`:9100`)
- [ ] 17 Loki categories present; 4 dashboards visible in the "SASE PoC" folder

---

## Related

- [Component: Telemetry Stack](../components/telemetry-stack.md)
- [Component: GNS3](../components/gns3.md)
- [Component: Wazuh](../components/wazuh.md)
- [Component: NetBird](../components/netbird.md)
- [Decision: Grafana over a custom UI](../decisions/grafana-vs-custom-ui.md)
- [Finding: RFC3164 vs RFC5424 syslog](../findings/rfc3164-vs-rfc5424-syslog.md)
- [Runbook 10: NATS JetStream](10-nats-jetstream.md)
- [Runbook 11: Wazuh SIEM](11-wazuh.md)

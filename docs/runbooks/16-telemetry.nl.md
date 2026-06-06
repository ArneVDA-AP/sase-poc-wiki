---
title: "Runbook: Telemetry Stack"
tags: [runbook, grafana, prometheus, loki, telemetry, docker]
---

# Runbook: Telemetry Stack

**Node(s):** mgmt01 (Docker, Grafana + Prometheus + Loki + forwarders), pop01 + dc01 (node_exporter)
**Vereisten:** Docker op mgmt01; [Runbook 10: NATS JetStream](10-nats-jetstream.nl.md) operationeel (de log forwarders consumeren de NATS bus)
**Status:** ✅ PoC-gevalideerd op de parallelle stack, sandbox-integratie pending

> **Scope.** Deze stack draait op de parallelle SASE_POC-stack (mgmt01) en is **nog niet in het sandbox-geheel geïntegreerd**. De security-observability van de sandbox is [Wazuh SIEM](../components/wazuh.nl.md), en dat blijft de source of truth. Deze runbook zet de aanvullende metrics/log-visualisatielaag op.

---

## Vereistenchecklist

- [ ] Docker en Docker Compose geïnstalleerd op mgmt01
- [ ] NATS JetStream operationeel op mgmt01 ([Runbook 10](10-nats-jetstream.nl.md)); de `LOGS` stream draagt de log feeds
- [ ] Wazuh-stack bereikbaar op mgmt01 (OpenSearch `localhost:9200`) voor de Wazuh forwarder ([Runbook 11](11-wazuh.nl.md))
- [ ] GNS3 REST API bereikbaar op `192.168.122.1:3080`
- [ ] SSH-toegang tot mgmt01 (en dc01, voor node_exporter)

---

## Stap 1: Layout van de Docker-stack op mgmt01

De stack staat in `/opt/sase-monitor/`. Maak de mappenstructuur aan voor de drie backends, de dashboards en de build contexts van de forwarders:

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

> **Valkuil: schrijf config-bestanden met Python, niet met bash heredocs.** Speciale tekens (`{}`, `${}`, `\n`) worden door bash heredocs verkeerd geïnterpreteerd en maken het bestand corrupt. Gebruik `sudo python3 -c "open(...).write(...)"` voor elk config-bestand.

---

## Stap 2: Prometheus configureren (host network)

Prometheus draait in `network_mode: host` zodat het exporters zowel binnen als buiten Docker kan scrapen.

> **Valkuil: Prometheus op host network heeft geen Docker DNS.** Het kan containernamen niet resolven. Target containers via hun bridge **gateway** IP (`172.x.x.1`) en host-exporters via `localhost`.

Scrape targets in `prometheus/prometheus.yml`:

| Job | Target | Opmerking |
|-----|--------|-----------|
| `prometheus` | `localhost:9090` | self-monitoring |
| `node-mgmt01` | `localhost:9100` | mgmt01 host metrics |
| `loki` | `172.x.0.2:3100` | **dit IP verschuift na een Loki-restart**, zie Stap 7 |
| `gns3` | `localhost:9456` | GNS3 node status |
| `node-dc01` | `10.0.0.100:9100` | momenteel DOWN (OPNsense blokkeert de scrape) |

Stel retentie in: `--storage.tsdb.retention.time=7d --storage.tsdb.retention.size=5GB`.

---

## Stap 3: Loki configureren

Kritische limits in `loki/loki-config.yml`:

```yaml
limits_config:
  retention_period: 168h            # 7 dagen
  reject_old_samples: false
  reject_old_samples_max_age: 720h  # accepteer tot 30 dagen oud
  creation_grace_period: 3h         # dekt de CEST/UTC-offset van OPNsense
```

> **Valkuil: zet geen `TZ=Europe/Brussels` op de Loki-container.** Dat vergiftigt streams met "entry too far behind" en het enige herstel is het Loki-volume wissen (destructief). De forwarders gebruiken `time.time()` (ingestijtijd) als Loki timestamp net om dit te vermijden. Zie [Component: Telemetry Stack](../components/telemetry-stack.nl.md).

---

## Stap 4: De log forwarders deployen

Logs komen in Loki via kleine Python-containers in plaats van Promtail (poort 514 is bezet door Wazuh, en de NATS bus bestaat al):

1. **`mon-nats-loki-forwarder`**: een durable consumer van `logs.>` op NATS JetStream. Leidt de Loki labels af uit het subject (`logs.pop01.firewall` → `device=pop01, category=firewall`). Draagt OPNsense syslog, Zeek, Caddy, NetBird container logs en M365.
2. **`mon-zeek-nats-publisher`**: tailt de Zeek spool (`/opt/zeek/spool/manager/*.log`) en publiceert naar `logs.mgmt01.zeek.{type}`.
3. **`mon-wazuh-loki-forwarder`**: pollt Wazuh OpenSearch (`level >= 5`) elke 15s en schrijft naar Loki `category=wazuh`. Zie [Runbook 11: Wazuh](11-wazuh.nl.md).
4. **`mon-gns3-exporter`**: pollt de GNS3 API elke 15s en exposeert `gns3_node_status{project,node,type,status}` (1=started, 0=stopped) op `:9456`.

> **Valkuil: dubbele Zeek-publicatie na reboot.** Een `zeek-to-nats.py` cron-job van een teamgenoot gebruikt dezelfde NATS credentials/subjects als `mon-zeek-nats-publisher`. Na een reboot kunnen beide draaien en entries dupliceren. Controleer `ps aux | grep zeek-to-nats` en `docker ps | grep zeek-nats`.

> **Valkuil: Zeek publisher kan in een warning loop blijven hangen.** Als hij bij de eerste scan een NATS-connectiefout krijgt, blijft hij loopen op "geen actieve Zeek logs gevonden", ook nadat de bestanden verschijnen. Fix met `docker restart mon-zeek-nats-publisher`.

---

## Stap 5: De NetBird API poller deployen

De NetBird container logs zijn te arm voor ZTNA-observability, dus een poller haalt elke 60s de REST API op. Zie [Component: NetBird](../components/netbird.nl.md).

1. Maak een NetBird **Personal Access Token** (PAT) aan, enkel via de dashboard UI (Team → Users → je eigen account). PATs beginnen met `nbp_`. Een setup key (UUID) is **geen** PAT.
2. De management container bindt poort 80 enkel op `127.0.0.1`, dus de poller moet de API **via Caddy over HTTPS** bereiken, niet rechtstreeks.
3. Bak de Caddy root CA in de poller-image zodat Python het `tls internal` certificaat vertrouwt:
   ```dockerfile
   COPY caddy-root.crt /usr/local/share/ca-certificates/caddy-root.crt
   RUN update-ca-certificates
   ```
4. Voeg `extra_hosts: ["netbird.sase.local:192.168.122.20"]` toe zodat de container de Caddy-hostname kan resolven.

> **Valkuil: het API-pad heeft geen versieprefix op v0.66.4.** De docs zeggen `/api/v1/peers`; deze versie geeft 404. Gebruik `/api/peers` en `/api/events`. Controleer altijd eerst met `curl`.

Builden en starten:

```bash
cd /opt/sase-monitor && sudo docker compose up -d --build netbird-api-poller
sleep 30 && sudo docker logs mon-netbird-api-poller --tail 20
```

De output moet peers tonen (`Peer gezien: WS-Rayan (connected)`) en events.

---

## Stap 6: node_exporter installeren op de gemonitorde hosts

Installeer op mgmt01 (en dc01) `node_exporter` v1.9.1 als systemd-service op `:9100`:

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

> **Opmerking: dc01 blijft DOWN tot OPNsense de scrape toelaat.** dc01 (`10.0.0.100`) zit achter OPNsense op de DC-LAN; pop01 blokkeert TCP van het WAN-segment naar `:9100`. Dit oplossen is een firewallregel-taak op pop01 (`pfctl -sr | grep 9100`), apart bijgehouden. Zie [Finding: DC-LAN-isolatie](../findings/dc-lan-isolation-route-acl.nl.md).

---

## Stap 7: De stack starten en verifiëren

```bash
cd /opt/sase-monitor && sudo docker compose up -d
sleep 15

echo "=== Grafana ==="    && curl -s http://localhost:3000/api/health
echo "=== Prometheus ===" && curl -s http://localhost:9090/-/healthy
echo "=== Loki ==="       && curl -s http://localhost:3100/ready
```

Controleer alle Prometheus targets:

```bash
curl -s http://localhost:9090/api/v1/targets | python3 -c "
import json,sys
for t in json.load(sys.stdin)['data']['activeTargets']:
    print(f'{t[\"health\"]:6s} {t[\"labels\"][\"job\"]:14s} {t.get(\"lastError\",\"\")[:50]}')"
```

> **Valkuil: Loki target DOWN na een restart = IP-verschuiving.** Docker wijst het `mon-loki` bridge-IP opnieuw toe bij een restart, waardoor de Prometheus `loki` target `connection refused` geeft. Fix:
> ```bash
> sudo docker inspect mon-loki | grep '"IPAddress"'
> sudo python3 -c "open('/opt/sase-monitor/prometheus/prometheus.yml','w').write(open('/opt/sase-monitor/prometheus/prometheus.yml').read().replace('OLD_IP','NIEUW_IP'))"
> sudo docker restart mon-prometheus
> ```
> Duurzame fix: Prometheus op `monitor-net` zetten en `mon-loki` als hostname targeten.

---

## Stap 8: Dashboards verifiëren in Grafana

Grafana provisiont vier dashboards (folder "SASE PoC") vanuit JSON-bestanden; ze overleven container rebuilds.

| Dashboard | Wat te bevestigen |
|-----------|-------------------|
| Node Status | GNS3 nodes tonen GROEN/ROOD; `gns3_node_status` geeft data terug |
| VM Metrics | mgmt01 CPU/RAM/disk/netwerk gevuld (dc01 leeg tot de opmerking bij Stap 6 opgelost is) |
| OPNsense Log Stream | Live logs lopen; Device/Component-dropdowns tonen pop01/mgmt01/wazuh |
| Stack Health | Alle Prometheus targets UP behalve `node-dc01` |

Bevestig de actieve Loki categories (verwacht 17):

```bash
curl -s -u admin:'<wachtwoord>' \
  "http://localhost:3000/api/datasources/proxy/uid/<LOKI_UID>/loki/api/v1/label/category/values" \
  | python3 -c "import json,sys; print(json.load(sys.stdin).get('data',[]))"
```

> **Opmerking: geprovisionde datasources/dashboards zijn read-only in de UI.** Wijzig de YAML/JSON-bestanden en herstart de container; UI-aanpassingen blijven niet bewaard.

---

## Checklist

- [ ] Grafana, Prometheus en Loki containers healthy op mgmt01
- [ ] Prometheus targets UP: `prometheus`, `node-mgmt01`, `loki`, `gns3` (dc01 DOWN verwacht)
- [ ] `mon-nats-loki-forwarder` consumeert `logs.>` → Loki
- [ ] `mon-zeek-nats-publisher` en `mon-wazuh-loki-forwarder` voeden Loki
- [ ] `mon-gns3-exporter` exposeert `gns3_node_status` op `:9456`
- [ ] `mon-netbird-api-poller` schrijft `category=netbird_api` (PAT geldig, Caddy CA vertrouwd)
- [ ] node_exporter actief op mgmt01 (`:9100`)
- [ ] 17 Loki categories aanwezig; 4 dashboards zichtbaar in de folder "SASE PoC"

---

## Gerelateerd

- [Component: Telemetry Stack](../components/telemetry-stack.nl.md)
- [Component: GNS3](../components/gns3.nl.md)
- [Component: Wazuh](../components/wazuh.nl.md)
- [Component: NetBird](../components/netbird.nl.md)
- [Beslissing: Grafana boven een custom UI](../decisions/grafana-vs-custom-ui.nl.md)
- [Finding: RFC3164 vs RFC5424 syslog](../findings/rfc3164-vs-rfc5424-syslog.nl.md)
- [Runbook 10: NATS JetStream](10-nats-jetstream.nl.md)
- [Runbook 11: Wazuh SIEM](11-wazuh.nl.md)

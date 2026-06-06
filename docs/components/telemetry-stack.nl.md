---
title: "Telemetry Stack (Grafana + Prometheus + Loki)"
tags: [grafana, prometheus, loki, telemetry, docker, nats-jetstream, netbird, wazuh, siem]
---

# Telemetry Stack (Grafana + Prometheus + Loki)

**Rol:** Observability-laag op de parallelle stack. Brengt metrics (Prometheus), logs (Loki) en SASE security telemetry samen in één Grafana "single pane of glass" voor de PoC-omgeving.  
**Status:** ✅ PoC-gevalideerd op de parallelle stack, sandbox-integratie pending  
**Versie:** Grafana 11.6.1, Prometheus (latest, gepind in compose), Loki 3.4.3  
**Config-locatie:** mgmt01, `/opt/sase-monitor/` (Docker Compose)

> **Sidenote, scope.** Deze stack is gebouwd en gevalideerd op de parallelle SASE_POC-stack (mgmt01), maar is **nog niet in het sandbox-geheel geïntegreerd**. De observability van de sandbox zelf is **Wazuh SIEM** (zie [Wazuh](wazuh.nl.md)), en dat blijft de source of truth voor security-alerting. De telemetry stack is een aanvullende monitoringlaag, geen vervanging van Wazuh.

---

## Hoe het werkt in deze stack

De telemetry stack beantwoordt de SASE-observability vraag: wie heeft toegang (NetBird peer events), wat doen ze (Zeek conn/dns/http), worden er dreigingen gedetecteerd (Wazuh/Suricata) en is de infrastructuur gezond (GNS3 node status plus node_exporter). In plaats van één tool per apparaat komt alles samen in één Grafana interface.

Achter Grafana zitten drie storage backends:

- **Prometheus**: pull-based metrics database. Draait in `network_mode: host` zodat het exporters kan scrapen die zowel binnen Docker leven (GNS3 exporter) als erbuiten (node_exporter op mgmt01 en dc01).
- **Loki**: lichtgewicht log store (een slankere optie dan Elasticsearch), met 7 dagen retentie.
- **Grafana**: visualisatie en dashboards, geprovisioned vanuit bestanden zodat dashboards container rebuilds overleven.

Logs komen in de huidige build niet via Promtail in Loki terecht. In plaats daarvan voeden een set kleine Python forwarder-containers Loki, en de meeste logbronnen lopen via de bestaande **NATS JetStream** bus (gebouwd door een teamgenoot) en niet via een aparte syslog-ontvanger. Zie [Architectuur: twee builds](#architectuur-twee-builds-poc-1a-vs-mgmt01) voor de reden.

### Containers (mgmt01-build)

| Container | Image | Netwerk | Functie |
|-----------|-------|---------|---------|
| `mon-grafana` | grafana/grafana:11.6.1 | monitor-net | Dashboard UI (`:3000`) |
| `mon-prometheus` | prom/prometheus | host | Metrics scraping |
| `mon-loki` | grafana/loki:3.4.3 | monitor-net | Log-opslag (`:3100`) |
| `mon-nats-loki-forwarder` | custom Python | monitor-net + NATS-net | Abonneert op `logs.>` in NATS, pusht naar Loki |
| `mon-gns3-exporter` | python:3.12-slim | host | GNS3 API → Prometheus (`:9456`) |
| `mon-zeek-nats-publisher` | custom Python | monitor-net + NATS-net | Tailt Zeek spool → NATS |
| `mon-wazuh-loki-forwarder` | custom Python | host | Wazuh OpenSearch (`level >= 5`) → Loki |
| `mon-netbird-api-poller` | custom Python | monitor-net + netbird-net | NetBird REST API → Loki |

Het forwarder-ontwerp is bewust container-per-bron, zodat één kapotte feed de andere niet kan stilleggen.

---

## Dashboards

Vier dashboards staan in de Grafana folder "SASE PoC", geprovisioned vanuit JSON-bestanden (`/opt/sase-monitor/grafana/dashboards/`). Geprovisionde dashboards en datasources zijn **read-only in de UI**. Wijzigingen moeten in de YAML/JSON-bestanden gebeuren plus een container restart.

| Dashboard | Datasource | Toont |
|-----------|------------|-------|
| **Node Status** (GNS3) | Prometheus | Eén stat panel per GNS3 node, GROEN = started / ROOD = stopped; tellers voor nodes started/stopped. Query: `gns3_node_status` |
| **VM Metrics** (Host Metrics) | Prometheus | CPU / RAM / disk I/O / netwerk voor mgmt01 (`node_exporter`). dc01 panels aanwezig maar leeg (target DOWN, zie gotchas) |
| **OPNsense Log Stream** (SASE Security Logs) | Loki | Live log stream met Device- en Component-dropdownfilters; firewall events; log volume per device. Query: `{job=~"sase-logs\|wazuh-alerts", device=~"$device", category=~"$category"}` |
| **Stack Health** | Prometheus | Prometheus target UP/DOWN, Loki ingestion rate, scrape duration. Query: `up{job='...'}` |

NetBird ZTNA-data verschijnt in het Security Logs dashboard onder `category=netbird_api`; Wazuh alerts onder `device=wazuh`. Een dedicated NetBird/ZTNA dashboard is een geplande uitbreiding, nog niet gebouwd.

---

## Log pipeline

In de mgmt01-build zijn er 17 actieve log categories in Loki, verdeeld over drie devices (pop01, mgmt01, wazuh):

```
caddy, clamav, firewall, m365, netbird, netbird_api, system, unbound,
wazuh, zeek_conn, zeek_dns, zeek_http, zeek_notice, zeek_ssh, zeek_ssl,
zeek_syslog, zeek_weird
```

De forwarders leiden de Loki labels af uit het NATS subject: `logs.pop01.firewall` wordt `device=pop01, category=firewall`. Een belangrijke designkeuze is het gebruik van `time.time()` (de Loki ingestijtijd) als log timestamp in plaats van de originele timestamp. OPNsense (FreeBSD) stuurt timestamps die `+00:00` claimen maar eigenlijk CEST zijn, wat Loki streams vergiftigt met "entry too far behind" weigeringen. Zie [Finding: RFC3164 vs RFC5424 syslog](../findings/rfc3164-vs-rfc5424-syslog.nl.md) voor het verwante syslog-formaatprobleem in de eerdere build.

De **NetBird API poller** verdient een aparte vermelding. De NetBird management container logt nauwelijks iets bruikbaars (~10 regels per uur aan ruis), dus de rijke ZTNA-data (welke peers verbonden zijn, `last_seen`, beleidswijzigingen, wie wat deed) wordt elke 60 seconden uit de NetBird REST API gehaald en naar Loki geschreven als `category=netbird_api`. Dit staat in contrast met de [NetBird](netbird.nl.md) container logs (`category=netbird`), die wel bewaard worden maar structureel te arm zijn voor observability.

---

## Integratiepunten

| Interface | Richting | Details |
|-----------|----------|---------|
| NATS JetStream (`logs.>`) | Inkomend | `mon-nats-loki-forwarder` is een durable consumer; brengt OPNsense syslog, Zeek, Caddy, NetBird en M365 in Loki |
| [GNS3](gns3.nl.md) REST API | Inkomend (poll) | `mon-gns3-exporter` pollt `192.168.122.1:3080` elke 15s → Prometheus metric `gns3_node_status` |
| node_exporter (`:9100`) | Inkomend (scrape) | mgmt01 (UP) en dc01 (DOWN, OPNsense blokkeert de scrape) |
| [Wazuh](wazuh.nl.md) OpenSearch (`:9200`) | Inkomend (poll) | `mon-wazuh-loki-forwarder` pollt `level >= 5` alerts → Loki `category=wazuh` |
| [NetBird](netbird.nl.md) REST API (`/api/peers`, `/api/events`) | Inkomend (poll) | `mon-netbird-api-poller` via Caddy HTTPS → Loki `category=netbird_api` |
| [Caddy](caddy.nl.md) | → browser | `grafana.sase.local` via Caddy reverse-proxiet (`tls internal`) voor NetBird-verbonden peers |

### Verhouding tot Wazuh SIEM

Wazuh en deze telemetry stack overlappen bewust, maar vervullen verschillende rollen. Wazuh is de **source of truth van de sandbox** voor security-detectie en CASB-enforcement: het parset, scoort en remedieert (optioneel). De telemetry stack **leest uit** Wazuh (de `wazuh-loki-forwarder` haalt `level >= 5` alerts in Loki) zodat Grafana security-alerts naast infrastructuur-metrics in één view kan tonen. De telemetry stack detecteert of enforcet niet; het aggregeert en visualiseert. Zie [Wazuh](wazuh.nl.md).

---

## Architectuur: twee builds (poc-1a vs mgmt01)

Er zijn **twee brondocumenten** die dit telemetriewerk beschrijven, op **twee verschillende hosts**. Ze spreken elkaar niet tegen (het tweede is een migratie van het eerste), maar wie de ruwe bronnen leest moet weten welke de huidige is.

| | **Build v1 (eerder)** | **Build v2 (huidig)** |
|---|---|---|
| Bron | `rayan_GNS3_Telemetry_Documentatie.md` (april 2026) | `SASE_Monitoring_Definitief_Verslag.md` (v3.0, mei 2026) |
| Host | poc-1a, `10.158.10.67` (Proxmox VM, buiten PoC scope) | mgmt01, `192.168.122.20` (binnen de PoC) |
| Log pipeline | netwerkapparaten → **rsyslog** relay → **Promtail** (TCP 1514) → Loki | NATS JetStream → Python forwarders → Loki |
| Syslog ingest | Promtail syslog-ontvanger; rsyslog converteert RFC3164→RFC5424 | Geen Promtail; geen poort-514-ontvanger (514 is bezet door Wazuh) |
| Reverse proxy | Nginx op `:8080` | Caddy (hergebruikt van NetBird) |
| Extra's | n.v.t. | NetBird API poller, Wazuh forwarder, Zeek publisher |

> ⚠️ **Tegenstrijdigheid (host/scope).** Het april-document legt de host vast als **poc-1a (`10.158.10.67`)** en het mei-document als **mgmt01 (`192.168.122.20`)**, met de expliciete vermelding dat de stack *gemigreerd* werd van poc-1a ("buiten PoC scope") naar mgmt01 ("binnen de PoC"). Volgens het authority-model van de wiki is de **latere verslag (mgmt01) de huidige staat**. De poc-1a-build staat hier enkel als de eerdere evolutie. Het mei-document merkt ook op dat "de wiki" mgmt01 als `.23` vermeldt, maar `.20` klopt voor het SASE_POC (team) project; let op dat de *sandbox* mgmt01 wél `192.168.122.23` is (zie [GNS3](gns3.nl.md)). Dit zijn twee verschillende projecten op dezelfde GNS3-host.

De april-build (poc-1a) gebruikte een Promtail syslog-ontvanger die niet overweg kon met het RFC3164-formaat dat netwerkapparaten sturen, en daarom werd een rsyslog relay toegevoegd. De mei-build (mgmt01) omzeilt dit volledig door op de bestaande NATS bus mee te liften. De RFC3164/RFC5424-les staat als [finding](../findings/rfc3164-vs-rfc5424-syslog.nl.md) gedocumenteerd omdat het een terugkerende valkuil is bij log pipelines, ook al loopt de huidige build er niet meer tegenaan.

---

## Bekende problemen / valkuilen

- **Loki bridge-IP verandert na restart (Prometheus target DOWN).** Docker wijst bridge-IPs dynamisch toe; na een herstart van `mon-loki` kan het IP verschuiven (bijvoorbeeld `.3` → `.2`), en gaat de Prometheus `loki` scrape target DOWN met `connection refused`. Fix: `docker inspect mon-loki` voor het nieuwe IP, `prometheus.yml` bijwerken, dan `docker restart mon-prometheus`. De duurzame fix is Prometheus op `monitor-net` zetten en `mon-loki` als hostname targeten, of een vast IP gebruiken.
- **Prometheus op host network kan containernamen niet resolven.** Prometheus op host network heeft geen Docker DNS en moet containers dus via hun bridge gateway IP (`172.x.x.1`) targeten, niet via de naam. Grafana (op `monitor-net`) bereikt Prometheus via het Docker gateway IP, niet via `localhost`.
- **dc01-metrics onbereikbaar.** De Prometheus target `node-dc01` (`10.0.0.100:9100`) is DOWN. OPNsense (pop01) blokkeert TCP van het WAN-segment naar de DC-LAN `:9100`. Een WAN-firewallregel werd aangemaakt maar matcht niet correct. Zie [Finding: DC-LAN-isolatie](../findings/dc-lan-isolation-route-acl.nl.md) voor hetzelfde DC-LAN-bereikbaarheidspatroon.
- **Loki "entry too far behind" / stream-vergiftiging.** Een eerdere `TZ=Europe/Brussels` instelling op Loki vergiftigde streams; herstel vereiste het wissen van het Loki data-volume (`docker volume rm sase-monitor_loki-data`), wat destructief is. Voortaan vermeden door `time.time()` als ingestijtijd te gebruiken.
- **Dubbele Zeek-publicatie na reboot.** Een `zeek-to-nats.py` cron-job van een teamgenoot en de `mon-zeek-nats-publisher` container gebruiken dezelfde NATS credentials op dezelfde subjects; na een reboot kunnen beide draaien en dubbele Zeek-entries produceren. Controleer `ps aux | grep zeek-to-nats` en `docker ps | grep zeek-nats` na elke reboot.
- **Docker vs UFW.** Docker manipuleert iptables direct en omzeilt UFW. In de v1-build waren enkel poort 8080 (Nginx) en 514/UDP (rsyslog) extern bereikbaar; de backends bleven op interne Docker-netwerken. Filtering op netwerkniveau wordt aan de Proxmox firewall overgelaten.

---

## Gerelateerd

- [Component: GNS3](gns3.nl.md)
- [Component: Wazuh](wazuh.nl.md)
- [Component: NetBird](netbird.nl.md)
- [Component: Caddy](caddy.nl.md)
- [Beslissing: Grafana boven een custom UI](../decisions/grafana-vs-custom-ui.nl.md)
- [Finding: RFC3164 vs RFC5424 syslog](../findings/rfc3164-vs-rfc5424-syslog.nl.md)
- [Runbook: Telemetry Stack](../runbooks/16-telemetry.nl.md)

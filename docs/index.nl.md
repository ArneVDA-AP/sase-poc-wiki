---
title: "Project Wiki"
tags: [sase, architecture]
---

# Project Wiki

Deze wiki documenteert de implementatie van een SASE (Secure Access Service Edge) proof-of-concept voor Atlascollege (een school met 4 000 gebruikers op beheerde Windows-apparaten, Intune-beheerd, Entra joined). De stack vervangt het traditionele perimetermodel door identiteitsgebaseerde, contextbewuste toegangscontrole met open-source componenten: NetBird (ZTNA), Squid (SWG), ClamAV + Python DLP (ICAP-inspectie), Suricata (IDS), ioc2rpz + Unbound (DNS threat intelligence), Entra ID (identiteit), NATS JetStream (event bus), Wazuh (SIEM) en een Python control daemon (real-time quarantaine). Alle componenten draaien op een GNS3-topologie gehost op Proxmox.

> Alle brondocumenten (verslagen, addenda, implementatiedocumenten) zijn beschikbaar op Microsoft Teams: **1A › project-wiki.rar**

[:material-sitemap: Open het functioneel schema &nearr;](demos/functioneel-schema.html){ .md-button .md-button--primary target=_blank }

Een interactief end-to-end-diagram van de SASE PoC: elke component, verkeersstroom en
policy-beslissingspunt. Opent full-width in een nieuw tabblad.

## Catalogus

| Pagina | Samenvatting |
|--------|--------------|
| **Overzicht** | |
| [Architectuur](overview/architecture.md) | Volledig systeem: nodes, netwerksegmenten, verkeersstromen, inspectiepipeline, trust boundaries |
| **Componenten** | |
| [Squid](components/squid.md) | Expliciete HTTP/HTTPS-proxy: WPAD/PAC-distributie, SSL Bump, URL-filtering, ICAP-orchestratie |
| [ClamAV/c-icap](components/clamav-cicap.md) | Malware-scanning + DLP Laag 1 (downloads) via ICAP RESPMOD; YARA-regels; StructuredDataDetection |
| [Python DLP](components/python-dlp.md) | Upload-DLP (ICAP REQMOD): Luhn, IBAN mod-97, BSN 11-proef, AWS-sleuteldetectie op multipart-bodies |
| [Suricata](components/suricata.md) | IDS op WAN (vtnet0) + LAN (vtnet1); Hyperscan; 79 620+ regels; IPS klaar voor fysieke hardware |
| [ioc2rpz](components/ioc2rpz.md) | DNS threat intelligence: URLhaus + ThreatFox → RPZ-zone → BIND → Unbound; 71 767 geblokkeerde domeinen |
| [NetBird](components/netbird.md) | WireGuard ZTNA-overlay: Zitadel + Entra ID IdP-keten; ACL-policies; DNS primaire nameserver |
| [Caddy](components/caddy.md) | WPAD PAC-bestandsserver, NetBird TLS-terminator, ioc2rpz GUI reverse proxy |
| [GNS3](components/gns3.md) | Lab-virtualisatieplatform: geneste QEMU/KVM op Proxmox; meerdere gebruikers; GNS3 vs EVE-NG |
| [VyOS](components/vyos.md) | SASE Gateway voor remote site (site01); Zero Trust Branch-model; NAT voor Site-LAN |
| [Identity Bridge](components/identity-bridge.md) | NetBird overlay-IP → Entra ID-groep (SWG↔ZTNA-koppeling) |
| [NATS JetStream](components/nats-jetstream.md) | Centrale event bus, verbindt detectiesilo's |
| [Control Daemon](components/control-daemon.md) | Threat scoring + real-time quarantaine via NetBird API |
| [Wazuh](components/wazuh.md) | SIEM: NATS-forwarder + pop01-agent + M365 Active Response |
| [Zitadel](components/zitadel.md) | OIDC IdP-broker: Entra ID → JWT group sync → NetBird |
| [Intune Endpoint Enforcement](components/intune-endpoint-enforcement.md) | MDM-config-push naar het endpoint: afgedwongen PAC, trusted-cert, firewall-block-set, DSCP/QoS, split-tunnel-routes |
| [Transparante Proxy](components/transparent-proxy.md) | Transparante (TPROXY) interceptie op linuxpop01 voor all-traffic capture (parallel-stack PoC). |
| [Cosmos](components/cosmos.md) | Identity-aware application gateway met per-app MFA (parallel-stack PoC). |
| [Zeek](components/zeek.md) | Network security monitor voor protocol-analyse en behavioral logging (parallel-stack PoC). |
| [RITA](components/rita.md) | Beaconing / C2 behavioral analysis over Zeek-logs, voedt RPZ (parallel-stack PoC). |
| [Telemetry Stack](components/telemetry-stack.md) | Grafana + Prometheus + Loki observability-laag (parallel-stack PoC). |
| **Concepten** | |
| [SASE](concepts/sase.md) | Vijf SASE-pijlers; onze open-source invulling; commerciële equivalenten |
| [Zero Trust](concepts/zero-trust.md) | Drie-gate model; never trust always verify; Gates 1–3 |
| [ICAP](concepts/icap.md) | REQMOD vs RESPMOD; Squid-orchestratie; waarom twee ICAP-servers |
| [SSL Bump](concepts/ssl-bump.md) | HTTPS-interceptie; SASE-PoC-CA; no-bump lijst |
| [WPAD/PAC](concepts/wpad-pac.md) | Browser proxy-autoconfiguratie; waarom transparante proxy faalt op wt0 |
| [RPZ](concepts/rpz.md) | DNS Response Policy Zones; zone-transferketen; NXDOMAIN-handhaving |
| [DLP](concepts/dlp.md) | Twee-laags DLP; algoritmische validatie vs patroonherkenning; upload vs download |
| [Identiteitsstroom](concepts/identity-flow.md) | Volledige identiteitsketen: Entra ID → Zitadel → NetBird → Identity Bridge → Squid |
| [Application Gateway](concepts/application-gateway.md) | Reverse proxy met identity gate en MFA als application-admission-laag. |
| [Behavioral Analysis](concepts/behavioral-analysis.md) | Beaconing / C2-detectie op gedrag in plaats van signature. |
| **Beslissingen** | |
| [WPAD/PAC vs Transparante Proxy](decisions/wpad-vs-transparent-proxy.md) | Waarom expliciete proxy (pf rdr werkt niet op wt0) |
| [Twee-laags DLP](decisions/two-layer-dlp.md) | ClamAV (downloads) + Python DLP (uploads), multipart-parseerprobleem |
| [ioc2rpz vs Unbound native](decisions/ioc2rpz-vs-unbound-native.md) | Waarom ioc2rpz voor feed-aggregatie in plaats van cron + zone-bestand |
| [BIND als TSIG-tussenstap](decisions/bind-tsig-intermediary.md) | Unbound 1.24.2 mist TSIG; BIND overbrugt het gat |
| [IDS vs IPS](decisions/ids-vs-ips.md) | Netmap IPS faalt op virtio-NICs, IDS-modus met policies klaar |
| [Suricata WAN+LAN](decisions/suricata-wan-lan.md) | Waarom vtnet0 + vtnet1 (wt0 toont 0 pakketjes in BPF) |
| [GNS3 vs EVE-NG](decisions/gns3-vs-eveng.md) | Multi-gebruikersvereiste; QCOW2-formaat; EVE-NG single-session beperking |
| [Zitadel als IdP-broker](decisions/zitadel-idp-broker.md) | Quickstart installeert Zitadel; Entra ID als externe IdP; CA viert nog steeds |
| [CA + Posture hybride (Drie-gate model)](decisions/ca-posture-hybrid.md) | Gate 1 (Entra ID CA) + Gate 2 (Intune compliance), complementair niet uitwisselbaar |
| [SD-WAN geschrapt (F12, F13, F14)](decisions/sdwan-descoped.md) | Klassiek IPsec + uCPE verwijderd (site-to-site tunnels in strijd met Zero Trust); QoS + failover heringevoerd onder het ZT-Branch-model; sitepc01 via NetBird-inschrijving |
| [NetBird Service PAT](decisions/netbird-service-pat.md) | Service-user PAT voor Identity Bridge API-auth (vermijdt NetBird issue #3127) |
| [NATS accounts auth](decisions/nats-accounts-auth.md) | `accounts{}`-model vereist voor JetStream API-toegang |
| [GroupSync Pad B](decisions/groupsync-pad-b.md) | Zitadel verwijdert `2ITCSC1A-`-prefix, schone persona-groepnamen |
| [CASB drie lagen](decisions/casb-three-layers.md) | Inline (Squid) + API (Wazuh) + Real-time (NATS+daemon) |
| [Beheerde apparaten scope](decisions/managed-devices-scope.md) | BYOD naar beheerde Windows-apparaten (lectoraatmandaat R11) |
| [ZT SD-WAN Branch](decisions/zt-sdwan-branch.md) | Zero Trust Branch vervangt klassieke IPsec SD-WAN |
| [Control Daemon scope](decisions/control-daemon-scope.md) | IDS-correlatie + proxy_block verwijderd uit threat scoring |
| [Cosmos Two-Layer ZTNA](decisions/cosmos-two-layer-ztna.md) | Waarom NetBird (device) plus Cosmos (application) twee ZTNA-lagen vormen. |
| [Hub vs Switch Visibility](decisions/hub-vs-switch-visibility.md) | Waarom een GNS3 Hub (geen Switch) voor volledige traffic-visibility naar Zeek. |
| [RITA RPZ Automation](decisions/rita-rpz-automation.md) | RITA-beacons als derde dynamische RPZ-threat-intel-feed. |
| [Grafana vs Custom UI](decisions/grafana-vs-custom-ui.md) | Waarom Grafana boven een custom React-UI voor telemetry. |
| **Bevindingen** | |
| [wt0 pf rdr beperking](findings/wt0-pf-rdr-limitation.md) | WireGuard Laag 3: pf rdr kan niet onderscheppen op wt0 |
| [pre-auth ssl-bump parameters](findings/pre-auth-ssl-bump-params.md) | Kale http_port zonder ssl-bump = geen inspectie op overlay-listener |
| [Squid clearlog vernietigt bestand](findings/squid-clearlog-destroys-file.md) | WebUI "Clear log" verwijdert in plaats van afkappen; gebruik `> access.log` |
| [StevenBlack incompatibel](findings/stevenblack-incompatible.md) | Hosts-formaat incompatibel met OPNsense Remote ACL; gebruik UT1 Toulouse |
| [Suricata interface default bug](findings/suricata-interface-default-bug.md) | `interface: default` vangt alleen vtnet0; expliciete declaraties vereist |
| [Suricata Netmap/virtio](findings/suricata-netmap-virtio.md) | Netmap IPS = 0 pakketjes op virtio-NICs |
| [Unbound geen TSIG](findings/unbound-no-tsig.md) | Unbound 1.24.2 mist TSIG voor zone-transfers (NLnetLabs issue #336) |
| [Unbound config-pad](findings/unbound-config-path.md) | `/var/unbound/etc/` wordt verwijderd bij herstart; gebruik `/usr/local/etc/unbound.opnsense.d/` |
| [ioc2rpz GUI JS bug](findings/ioc2rpz-gui-js-bug.md) | Ontbrekende e.preventDefault() in signIn; sed-fix na containerstart |
| [pyicap collections bug](findings/pyicap-collections-bug.md) | `collections.Callable` verwijderd in Python 3.10; sed-patch in Dockerfile |
| [iptables FORWARD-volgorde](findings/iptables-forward-ordering.md) | libvirt REJECT vóór toegevoegde regels; gebruik altijd `-I FORWARD 1` |
| [NetBird primaire nameserver](findings/netbird-primary-nameserver.md) | Lege match-domains vereist zodat RPZ externe domeinen dekt |
| [NetBird config nul bytes](findings/netbird-config-zero-bytes.md) | FreeBSD UFS + QEMU SIGKILL = 0-byte config.json; back-up na elke sessie |
| [Docker volume hermaak](findings/docker-volume-recreation.md) | `docker compose restart` past volume-mount wijzigingen niet toe; gebruik `up -d` |
| [curl --ssl-no-revoke op Windows](findings/curl-ssl-no-revoke.md) | Schannel CRL-controle faalt op SASE-PoC-CA; voeg `--ssl-no-revoke` toe aan alle proxytests vanaf Windows |
| [Suricata connection pooling](findings/suricata-connection-pooling.md) | Squid hergebruikt upstream TCP-verbindingen; een Suricata-alert per SID per flow is correct, geen suppressie |
| [NetBird issue #3127](findings/netbird-issue-3127.md) | Management API weigert peer-level groepswijzigingen; service-user PAT als workaround |
| [Squid overlay bind race](findings/squid-overlay-bind-race.md) | Squid overlay-listener verloren bij herstart (`commBind EADDRNOTAVAIL` race) |
| [Overlay IP-instabiliteit](findings/overlay-ip-instability.md) | NetBird overlay-IP's kunnen wijzigen; Identity Bridge moet stale cache afhandelen |
| [NATS store dir](findings/nats-store-dir.md) | JetStream-opslag moet op persistent Docker-volume staan |
| [Wazuh CPU glibc](findings/wazuh-cpu-glibc.md) | CPU-piek bij opstarten door glibc-compatibiliteit |
| [GNS3-host RAM-overcommit](findings/gns3-host-ram-overcommit.md) | poc-1a committeert guest-RAM over met 0 swap; een geheugenpiek riskeert dat de OOM-killer een QEMU-guest neerhaalt |
| [Wazuh dashboard airgate](findings/wazuh-dashboard-airgate.md) | Dashboard Offline door lege UUID na down -v, opgelost via in-place 4.14.5 GA bump |
| [NetBird JWT allow-groups lockout](findings/netbird-jwt-allow-groups-lockout.md) | JWT allow-groups inschakelen kan alle gebruikers buitensluiten bij verkeerde configuratie |
| [DC-LAN isolatie route ACL](findings/dc-lan-isolation-route-acl.md) | NetBird Networks + ACL vereist voor DC-LAN-isolatie |
| [Cosmos hostname/OAuth-limiet](findings/cosmos-hostname-oauth.md) | Hostname-op-IP beperkt Cosmos cross-domain OAuth/SSO. |
| [DC-segment mirror-limiet](findings/dc-segment-mirror-limit.md) | FreeBSD kan niet kernel-mirroren; DC-binnensegment-visibility is partieel. |
| [VyOS GRE two-step commit](findings/vyos-gre-two-step-commit.md) | VyOS GRE-tunnel en mirror vereisen twee aparte commits. |
| [RFC3164 vs RFC5424 syslog](findings/rfc3164-vs-rfc5424-syslog.md) | Apparaten sturen RFC3164, tools verwachten RFC5424; rsyslog-relay overbrugt dit. |
| **Testen** | |
| [Acceptatietests (F1–F15)](testing/acceptance-tests.nl.md) | Volledige F1–F15-statustabel, testcommando's, werkelijke output, dekking per pijler, geplande tests |
| [Aanvals- en bypass-scenario's](testing/attack-scenarios.nl.md) | Demovalidatie-scenario's per SASE-pijler: ZTNA, SWG, CASB, FWaaS/IDS, DNS, NATS-handhaving |
| [Demo-script (per rubric)](testing/demo-script.nl.md) | Interactief presentatiescript dat elk rubric-criterium koppelt aan test, bewijs, screencap en voiceover |
| [Testblok Transparante Proxy](testing/transparent-proxy-tests.nl.md) | T-TP / T-ME-validatie van de transparent-proxy-diagnostiek en managed enforcement. |
| **Runbooks** | |
| [Runbook-overzicht](runbooks/index.md) | Bouwvolgorde, afhankelijkheidsgraph, stapsgewijze handleidingen |
| [01: Labomgeving](runbooks/01-lab-environment.md) | Proxmox VM, GNS3 Server, topologie, IP-adressering, snapshots |
| [02: ZTNA Overlay](runbooks/02-ztna-overlay.md) | NetBird + Zitadel + Entra ID, groepen, ACL's, exit node, DNS |
| [03: Proxy & WPAD](runbooks/03-proxy-wpad.md) | Squid expliciete proxy, WPAD/PAC via Caddy, SSL Bump, URL-filtering |
| [04: Malware & DLP](runbooks/04-malware-dlp.md) | ClamAV/c-icap RESPMOD, YARA-regels, Python DLP REQMOD |
| [05: IDS](runbooks/05-ids.md) | Suricata op WAN+LAN, Hyperscan, 79.620+ regels |
| [06: DNS Threat Intel](runbooks/06-dns-threat-intel.md) | ioc2rpz, BIND TSIG, Unbound RPZ, 71.767 records |
| [07: Access Policy](runbooks/07-access-policy.md) | Conditional Access, posture checks, validatiescenario's |
| [08: GroupSync](runbooks/08-groupsync.md) | JWT group sync, Entra ID token config, Zitadel Actions |
| [09: Identity Bridge](runbooks/09-identity-bridge.md) | FastAPI overlay-IP → persona-groep, Squid external_acl |
| [10: NATS JetStream](runbooks/10-nats-jetstream.md) | Event bus, producers, Control Daemon, Redis |
| [11: Wazuh](runbooks/11-wazuh.md) | SIEM-stack, NATS-forwarder, M365 Active Response |
| [13: Cosmos](runbooks/13-cosmos.md) | Cosmos-install en per-app MFA (parallelle stack). |
| [14: Zeek/RITA](runbooks/14-zeek-rita.md) | Hub/GRE-setup, Zeek-cluster, RITA-beacon-analyse (parallelle stack). |
| [15: RITA naar RPZ](runbooks/15-rita-rpz-integration.md) | Geautomatiseerde RITA naar ioc2rpz naar BIND naar Unbound-feed (parallelle stack). |
| [16: Telemetry](runbooks/16-telemetry.md) | Grafana / Prometheus / Loki-stack deployen (parallelle stack). |

---

## Snelle referentie

**Belangrijke poorten:**

| Service | Adres | Poort |
|---------|-------|-------|
| Squid proxy | `100.70.154.79` | 3128 |
| Unbound DNS | `100.70.154.79` | 53 |
| BIND (TSIG secundair) | `127.0.0.1` | 53530 |
| ioc2rpz | `192.168.122.23` | 53 |
| Python DLP ICAP | `192.168.122.23` | 1345 |
| ClamAV/c-icap | `127.0.0.1` | 1344 |
| NetBird Dashboard | `netbird.sandbox.local` | 443 |
| ioc2rpz GUI | `ioc2rpz.sandbox.local` | 443 |
| WPAD PAC-bestand | `wpad.sandbox.local` | 80 |

**Belangrijke IP-adressen:**

| Node | WAN | Overlay |
|------|-----|---------|
| pop01 | `192.168.122.13` | `100.70.154.79` |
| mgmt01 | `192.168.122.23` | `100.70.135.241` |
| mobile01 | n.v.t. | `100.70.95.98` |
| dc01 | n.v.t. | `10.0.0.100` |
| GNS3 host | `10.158.10.67` | n.v.t. |

**Gate-status:**

| Gate | Status | Technologie |
|------|--------|-------------|
| Gate 1 (Identiteit) | ✅ Operationeel | Entra ID Conditional Access (5 policies) |
| Gate 2 (Apparaat) | ✅ Operationeel | Intune device compliance (Report-only tot demo) |
| Gate 3 (Inhoud) | ✅ Operationeel | Squid + ClamAV + Python DLP + Unbound RPZ + Suricata |

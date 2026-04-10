---
title: "Project Wiki"
tags: [sase, architecture]
---

# Project Wiki

Deze wiki documenteert de implementatie van een SASE (Secure Access Service Edge) proof-of-concept voor Atlascollege — een school met 4000+ BYOD-studenten. De stack vervangt het traditionele perimetermodel door identiteitsgebaseerde, contextbewuste toegangscontrole met open-source componenten: NetBird (ZTNA), Squid (SWG), ClamAV + Python DLP (ICAP-inspectie), Suricata (IDS), ioc2rpz + Unbound (DNS threat intelligence) en Entra ID (identiteit). Alle componenten draaien op een GNS3-topologie gehost op Proxmox.

## Catalogus

| Pagina | Samenvatting |
|--------|--------------|
| **Overzicht** | |
| [Architectuur](overview/architecture.md) | Volledig systeem: nodes, netwerksegmenten, verkeersstromen, inspectiepipeline, vertrouwensgrenzen |
| **Componenten** | |
| [Squid](components/squid.md) | Expliciete HTTP/HTTPS-proxy — WPAD/PAC-distributie, SSL Bump, URL-filtering, ICAP-orchestratie |
| [ClamAV/c-icap](components/clamav-cicap.md) | Malware-scanning + DLP Laag 1 (downloads) via ICAP RESPMOD; YARA-regels; StructuredDataDetection |
| [Python DLP](components/python-dlp.md) | Upload-DLP (ICAP REQMOD) — Luhn, IBAN mod-97, BSN 11-proef, AWS-sleuteldetectie op multipart-bodies |
| [Suricata](components/suricata.md) | IDS op WAN (vtnet0) + LAN (vtnet1); Hyperscan; 79 620+ regels; IPS klaar voor fysieke hardware |
| [ioc2rpz](components/ioc2rpz.md) | DNS threat intelligence — URLhaus + ThreatFox → RPZ-zone → BIND → Unbound; 71 767 geblokkeerde domeinen |
| [NetBird](components/netbird.md) | WireGuard ZTNA-overlay — Zitadel + Entra ID IdP-keten; ACL-policies; DNS primaire nameserver |
| [Caddy](components/caddy.md) | WPAD PAC-bestandsserver, NetBird TLS-terminator, ioc2rpz GUI reverse proxy |
| [GNS3](components/gns3.md) | Lab-virtualisatieplatform — geneste QEMU/KVM op Proxmox; meerdere gebruikers; GNS3 vs EVE-NG |
| [VyOS](components/vyos.md) | SD-WAN-gateway voor remote site (site01); NAT voor Site-LAN |
| **Concepten** | |
| [SASE](concepts/sase.md) | Vijf SASE-pijlers; onze open-source invulling; commerciële equivalenten |
| [Zero Trust](concepts/zero-trust.md) | Drie-gate model; never trust always verify; Gates 1–3 |
| [ICAP](concepts/icap.md) | REQMOD vs RESPMOD; Squid-orchestratie; waarom twee ICAP-servers |
| [SSL Bump](concepts/ssl-bump.md) | HTTPS-interceptie; SASE-PoC-CA; no-bump lijst |
| [WPAD/PAC](concepts/wpad-pac.md) | Browser proxy-autoconfiguratie; waarom transparante proxy faalt op wt0 |
| [RPZ](concepts/rpz.md) | DNS Response Policy Zones; zone-transferketen; NXDOMAIN-handhaving |
| [DLP](concepts/dlp.md) | Twee-laags DLP; algoritmische validatie vs patroonherkenning; upload vs download |
| **Beslissingen** | |
| [WPAD/PAC vs Transparante Proxy](decisions/wpad-vs-transparent-proxy.md) | Waarom expliciete proxy — pf rdr werkt niet op wt0 |
| [Twee-laags DLP](decisions/two-layer-dlp.md) | ClamAV (downloads) + Python DLP (uploads) — multipart-parseerprobleem |
| [ioc2rpz vs Unbound native](decisions/ioc2rpz-vs-unbound-native.md) | Waarom ioc2rpz voor feed-aggregatie in plaats van cron + zone-bestand |
| [BIND als TSIG-tussenstap](decisions/bind-tsig-intermediary.md) | Unbound 1.24.2 mist TSIG — BIND overbrugt het gat |
| [IDS vs IPS](decisions/ids-vs-ips.md) | Netmap IPS faalt op virtio-NICs — IDS-modus met policies klaar |
| [Suricata WAN+LAN](decisions/suricata-wan-lan.md) | Waarom vtnet0 + vtnet1 — wt0 toont 0 pakketjes in BPF |
| [GNS3 vs EVE-NG](decisions/gns3-vs-eveng.md) | Multi-gebruikersvereiste; QCOW2-formaat; EVE-NG single-session beperking |
| [Zitadel als IdP-broker](decisions/zitadel-idp-broker.md) | Quickstart installeert Zitadel; Entra ID als externe IdP; CA viert nog steeds |
| [CA + Posture hybride (Drie-gate model)](decisions/ca-posture-hybrid.md) | Gate 1 (Entra ID CA) + Gate 2 (posture) — complementair niet uitwisselbaar; BYOD Intune-gat |
| **Bevindingen** | |
| [wt0 pf rdr beperking](findings/wt0-pf-rdr-limitation.md) | WireGuard Laag 3 — pf rdr kan niet onderscheppen op wt0 |
| [pre-auth ssl-bump parameters](findings/pre-auth-ssl-bump-params.md) | Kale http_port zonder ssl-bump = geen inspectie op overlay-listener |
| [Squid clearlog vernietigt bestand](findings/squid-clearlog-destroys-file.md) | WebUI "Clear log" verwijdert in plaats van afkapt; gebruik `> access.log` |
| [StevenBlack incompatibel](findings/stevenblack-incompatible.md) | Hosts-formaat incompatibel met OPNsense Remote ACL — gebruik UT1 Toulouse |
| [Suricata interface default bug](findings/suricata-interface-default-bug.md) | `interface: default` vangt alleen vtnet0 — expliciete declaraties vereist |
| [Suricata Netmap/virtio](findings/suricata-netmap-virtio.md) | Netmap IPS = 0 pakketjes op virtio-NICs |
| [Unbound geen TSIG](findings/unbound-no-tsig.md) | Unbound 1.24.2 mist TSIG voor zone-transfers (NLnetLabs issue #336) |
| [Unbound config-pad](findings/unbound-config-path.md) | `/var/unbound/etc/` wordt verwijderd bij herstart — gebruik `/usr/local/etc/unbound.opnsense.d/` |
| [ioc2rpz GUI JS bug](findings/ioc2rpz-gui-js-bug.md) | Ontbrekende e.preventDefault() in signIn — sed-fix na containerstart |
| [pyicap collections bug](findings/pyicap-collections-bug.md) | `collections.Callable` verwijderd in Python 3.10 — sed-patch in Dockerfile |
| [iptables FORWARD-volgorde](findings/iptables-forward-ordering.md) | libvirt REJECT vóór toegevoegde regels — gebruik altijd `-I FORWARD 1` |
| [NetBird primaire nameserver](findings/netbird-primary-nameserver.md) | Lege match-domains vereist zodat RPZ externe domeinen dekt |
| [NetBird config nul bytes](findings/netbird-config-zero-bytes.md) | FreeBSD UFS + QEMU SIGKILL = 0-byte config.json — back-up na elke sessie |
| [Docker volume hermaak](findings/docker-volume-recreation.md) | `docker compose restart` past volume-mount wijzigingen niet toe — gebruik `up -d` |

---

## Snelle referentie

**Belangrijke poorten:**

| Service | Adres | Poort |
|---------|-------|-------|
| Squid proxy (BYOD) | `100.70.154.79` | 3128 |
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
| mobile01 | — | `100.70.95.98` |
| dc01 | — | `10.0.0.100` |
| GNS3 host | `10.158.10.67` | — |

**Gate-status:**

| Gate | Status | Technologie |
|------|--------|-------------|
| Gate 1 — Identiteit | ⚠️ Gepland | Entra ID Conditional Access (4 policies in Addendum E) |
| Gate 2 — Apparaat | ⚠️ Gepland | NetBird Posture Checks (Addendum E) |
| Gate 3 — Inhoud | ✅ Operationeel | Squid + ClamAV + Python DLP + Unbound RPZ + Suricata |

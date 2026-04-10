---
title: "Wijzigingslog"
tags: [architecture]
---

# Wijzigingslog

Toevoeg-alleen log van wiki-wijzigingen.

---

## 2026-04-10 — Initiële ingest (volledige wiki-opbouw)

**Aangemaakte bestanden:**

### Overzicht
- `overview/architecture.md` — Volledige systeemarchitectuur: knooppunten, segmenten, opsplitsing control/data plane, vier verkeersstromen, inspectiepijplijn-tabel, vertrouwensgrenzen

### Componenten
- `components/squid.md` — Expliciete proxy, WPAD/PAC, SSL Bump, URL-filtering, ICAP-orkestratie
- `components/clamav-cicap.md` — Malware-scanning + DLP-laag 1, YARA-regels, StructuredDataDetection
- `components/python-dlp.md` — Upload-DLP ICAP REQMOD, pyicap-patch, multipart-parsing
- `components/suricata.md` — IDS op WAN+LAN, Hyperscan, custom.yaml, validatieresultaten
- `components/ioc2rpz.md` — ioc2rpz + BIND + Unbound RPZ-keten, 71 767 records, twee feeds
- `components/netbird.md` — WireGuard ZTNA-overlay, Zitadel + Entra ID, ACL-beleid, DNS
- `components/caddy.md` — WPAD-server, NetBird TLS-terminator, ioc2rpz GUI-proxy
- `components/gns3.md` — GNS3 op Proxmox, topologie, IP-tabel, snapshots, operationele valkuilen
- `components/vyos.md` — VyOS SD-WAN-gateway (stub — onvoldoende brondetail voor volledige configuratie)

### Concepten
- `concepts/sase.md` — Vijf SASE-pijlers, PoC-mapping, commerciële equivalenten
- `concepts/zero-trust.md` — Driepoortmodel, aanvullende poorten, principemapping
- `concepts/icap.md` — REQMOD vs. RESPMOD, Squid-orkestratie, twee-server-onderbouwing
- `concepts/ssl-bump.md` — HTTPS-interceptie, SASE-PoC-CA, no-bump-lijst, pre-auth ssl-bump
- `concepts/wpad-pac.md` — PAC-bestand ontdekkingsketen, transparante proxy mislukt, expliciete modus
- `concepts/rpz.md` — RPZ-zonetransferketens, NXDOMAIN-afdwinging, DNS-bereik
- `concepts/dlp.md` — Twee-laags DLP, algoritmische validatie, YARA-regelbereik

### Beslissingen
- `decisions/wpad-vs-transparent-proxy.md` — pf rdr mislukt op wt0 → WPAD/PAC
- `decisions/two-layer-dlp.md` — ClamAV multipart-kloof → Python DLP REQMOD
- `decisions/ioc2rpz-vs-unbound-native.md` — Feed-aggregatie → ioc2rpz Docker
- `decisions/bind-tsig-intermediary.md` — Unbound geen TSIG → BIND 9.20 als tussenpersoon
- `decisions/ids-vs-ips.md` — Netmap/virtio-incompatibiliteit → IDS-modus
- `decisions/suricata-wan-lan.md` — wt0 BPF 0 paketten → vtnet0+vtnet1
- `decisions/gns3-vs-eveng.md` — Meerdere-gebruikers-vereiste, QCOW2-formaat → GNS3
- `decisions/zitadel-idp-broker.md` — Quickstart installeert Zitadel, Entra ID als externe IdP
- `decisions/ca-posture-hybrid.md` — Intune-kloof op BYOD → hybride CA + postuur driepoortmodel

### Bevindingen
- `findings/wt0-pf-rdr-limitation.md`
- `findings/pre-auth-ssl-bump-params.md`
- `findings/squid-clearlog-destroys-file.md`
- `findings/stevenblack-incompatible.md`
- `findings/suricata-interface-default-bug.md`
- `findings/suricata-netmap-virtio.md`
- `findings/unbound-no-tsig.md`
- `findings/unbound-config-path.md`
- `findings/ioc2rpz-gui-js-bug.md`
- `findings/pyicap-collections-bug.md`
- `findings/iptables-forward-ordering.md`
- `findings/netbird-primary-nameserver.md`
- `findings/netbird-config-zero-bytes.md`
- `findings/docker-volume-recreation.md`

### Index en log
- `index.md` — Volledige catalogus, snelle naslag (poorten, IP's, poortstatus)
- `log.md` — Dit bestand

**Verwerkte brondocumenten:** Verslag18–27 (10 sessieverslagen), Doc1–Doc7 (7 implementatiedocumenten), SASE_Architectuur_Overzicht.md (1 architectuuroverzicht)

**Totaal aantal aangemaakte pagina's:** 37

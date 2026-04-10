---
title: "Change Log"
tags: [architecture]
---

# Change Log

Append-only log of wiki changes.

---

## 2026-04-10 — Initial ingest (full wiki build)

**Files created:**

### Overview
- `overview/architecture.md` — Full system architecture: nodes, segments, control/data plane split, four traffic flows, inspection pipeline table, trust boundaries

### Components
- `components/squid.md` — Explicit proxy, WPAD/PAC, SSL Bump, URL filtering, ICAP orchestration
- `components/clamav-cicap.md` — Malware scanning + DLP Layer 1, YARA rules, StructuredDataDetection
- `components/python-dlp.md` — Upload DLP ICAP REQMOD, pyicap patch, multipart parsing
- `components/suricata.md` — IDS on WAN+LAN, Hyperscan, custom.yaml, validation results
- `components/ioc2rpz.md` — ioc2rpz + BIND + Unbound RPZ chain, 71 767 records, two feeds
- `components/netbird.md` — WireGuard ZTNA overlay, Zitadel + Entra ID, ACL policies, DNS
- `components/caddy.md` — WPAD server, NetBird TLS terminator, ioc2rpz GUI proxy
- `components/gns3.md` — GNS3 on Proxmox, topology, IP table, snapshots, operational gotchas
- `components/vyos.md` — VyOS SD-WAN gateway (stub — insufficient source detail for full config)

### Concepts
- `concepts/sase.md` — Five SASE pillars, PoC mapping, commercial equivalents
- `concepts/zero-trust.md` — Three-gate model, complementary gates, principles mapping
- `concepts/icap.md` — REQMOD vs RESPMOD, Squid orchestration, two-server rationale
- `concepts/ssl-bump.md` — HTTPS interception, SASE-PoC-CA, no-bump list, pre-auth ssl-bump
- `concepts/wpad-pac.md` — PAC file discovery chain, transparent proxy failure, explicit mode
- `concepts/rpz.md` — RPZ zone transfer chain, NXDOMAIN enforcement, DNS scope
- `concepts/dlp.md` — Two-layer DLP, algorithmic validation, YARA rules scope

### Decisions
- `decisions/wpad-vs-transparent-proxy.md` — pf rdr fails on wt0 → WPAD/PAC
- `decisions/two-layer-dlp.md` — ClamAV multipart gap → Python DLP REQMOD
- `decisions/ioc2rpz-vs-unbound-native.md` — Feed aggregation → ioc2rpz Docker
- `decisions/bind-tsig-intermediary.md` — Unbound no TSIG → BIND 9.20 intermediary
- `decisions/ids-vs-ips.md` — Netmap/virtio incompatibility → IDS mode
- `decisions/suricata-wan-lan.md` — wt0 BPF 0 packets → vtnet0+vtnet1
- `decisions/gns3-vs-eveng.md` — Multi-user requirement, QCOW2 format → GNS3
- `decisions/zitadel-idp-broker.md` — Quickstart installs Zitadel, Entra ID as external IdP
- `decisions/ca-posture-hybrid.md` — Intune gap on BYOD → hybrid CA + posture three-gate model

### Findings
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

### Index and log
- `index.md` — Full catalog, quick reference (ports, IPs, gate status)
- `log.md` — This file

**Source documents ingested:** Verslag18–27 (10 session reports), Doc1–Doc7 (7 implementation documents), SASE_Architectuur_Overzicht.md (1 architecture overview)

**Total pages created:** 37

---

## 2026-04-10 — Lint pass

**Files updated:**

- `tags.md` — Populated controlled tag vocabulary (was empty stub): 4 page-type tags, 12 component tags, 15 protocol/technology tags, 5 security concept tags
- `components/vyos.md` — Removed TODO stub comment (VyOS is intentionally a topology-only stub; no source detail exists for config)

**Lint findings resolved:** 2 of 3 actionable items from lint report.  
**Deferred:** WireGuard concept page — WireGuard is NetBird's transport protocol; all WireGuard content belongs in `components/netbird.md`.

---

## 2026-04-10 — Review fixes (accuracy + coherence + schema)

**Files updated:**

- `components/caddy.md` — **Critical:** replaced incorrect `isInNet()` PAC file with actual deployed `shExpMatch()` version; corrected WPAD HTTP-only claim (WinHTTP tries HTTPS first — both blocks required); replaced incorrect ioc2rpz reverse proxy target (`ioc2rpz-ioc2rpz-gui-1:80` → `https://192.168.122.23:8444` with `tls_insecure_skip_verify`); added missing HTTPS WPAD Caddyfile block; added `caddy` and `wpad` tags to frontmatter
- `overview/architecture.md` — Added DC-LAN egress inspection gap to trust boundaries table with explanatory note; added ICAP `bypass=on` fail-open row to trust boundaries
- `decisions/two-layer-dlp.md` — Added Related section with back-links to component and concept pages
- `decisions/bind-tsig-intermediary.md` — Added Related section with back-links to ioc2rpz component and RPZ concept
- `concepts/sase.md` — Qualified inline CASB status from "Operational" to "Partial" (URL filtering + DLP only, no app-level controls)

**Review findings resolved:** 4 critical, 3 minor, 2 suggestions. Source-verified against `raw/Doc1_Squid_WPAD_PAC.md`, `raw/Doc4_DNS_Threat_Intelligence.md`, `raw/Verslag19.md`, and `raw/Verslag24.md`.

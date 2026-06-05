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

---

## 2026-04-10 — Runbooks added (deployment guides)

**Problem addressed:** The wiki was structured as a modular reference (components, concepts, decisions, findings) — excellent for lookups, but missing the linear step-by-step deployment trails from the original raw implementation documents. An engineer rebuilding the system had to hop between pages to reconstruct the build order.

**Files created:**

### Runbooks
- `runbooks/index.md` — Build order, dependency graph, runbook catalog
- `runbooks/01-lab-environment.md` — Proxmox VM, GNS3 Server, topology, IP addressing, snapshots (from Doc5)
- `runbooks/02-ztna-overlay.md` — NetBird + Zitadel + Entra ID, 15 steps, groups, ACLs, exit node, DNS (from Doc6)
- `runbooks/03-proxy-wpad.md` — Squid explicit proxy, WPAD/PAC via Caddy, SSL Bump, URL filtering, 10 steps (from Doc1)
- `runbooks/04-malware-dlp.md` — ClamAV/c-icap RESPMOD, YARA rules, Python DLP REQMOD, 7 steps (from Doc2)
- `runbooks/05-ids.md` — Suricata on WAN+LAN, Hyperscan, 79,620+ rules, 6 steps (from Doc3)
- `runbooks/06-dns-threat-intel.md` — ioc2rpz, BIND TSIG intermediary, Unbound RPZ, 71,767 records, 7 steps (from Doc4)
- `runbooks/07-access-policy.md` — Conditional Access (4 policies), posture checks, 5 validation scenarios, planned (from Doc7)

**Files updated:**

- `index.md` — Added Runbooks section to catalog table (8 entries)
- `overview/architecture.md` — Added link to runbooks in Related section
- `components/gns3.md` — Added runbook link to Related
- `components/netbird.md` — Added runbook links to Related
- `components/squid.md` — Added runbook link to Related
- `components/clamav-cicap.md` — Added runbook link to Related
- `components/python-dlp.md` — Added runbook link to Related
- `components/suricata.md` — Added runbook link to Related
- `components/ioc2rpz.md` — Added runbook link to Related
- `components/caddy.md` — Added runbook links to Related

**Total new pages:** 8 runbook pages. **Total updated pages:** 10.

---

## 2026-04-10 — Accuracy and completeness pass

**Problem addressed:** A full cross-reference audit compared all 18 raw source files against all 51 wiki pages. The audit found 7 factual inaccuracies and ~40 technical gaps where the raw sources contained details not captured in the wiki.

**Inaccuracies fixed (7):**

- `findings/pre-auth-ssl-bump-params.md` — Corrected: pre-auth include IS visible in `squid -k parse` (only invisible in `grep http_port squid.conf`)
- `findings/suricata-interface-default-bug.md` — Corrected: vtnet1 DID have a BPF fd open; Suricata wasn't routing packets to it
- `findings/ioc2rpz-gui-js-bug.md` — Corrected: browser displays "Unknown error!!!" (not silent failure); added permanent fix options
- `decisions/ca-posture-hybrid.md` — Corrected Policy 4 name: `SASE-PoC-Risk-Block` (was `SASE-PoC-Sign-in-Risk`)
- `decisions/ioc2rpz-vs-unbound-native.md` — Added missing `-p 953` to `rndc retransfer` command
- `decisions/bind-tsig-intermediary.md` — Added missing `-p 953` to `rndc retransfer` command
- `decisions/ids-vs-ips.md` — Replaced placeholder "Bevinding 23.x" with 23.3/23.4/23.5

**Gaps filled (22 files updated):**

- `concepts/wpad-pac.md` — Added NRPT/auto-discovery non-functional explanation; DHCP option 252 production path
- `concepts/zero-trust.md` — Added explicit gate-to-principle mapping (Verify Explicitly identity/device, Assume Breach)
- `components/squid.md` — Added `--management-url` mandatory gotcha; `wt0` IPv4 Configuration Type: None; handbook NAT rule irrelevant
- `components/netbird.md` — Added distribution groups vs ACL distinction; WireGuard port 51820; `process_check` platform limitations
- `components/python-dlp.md` — Added code review findings (OOM, log injection, exception handling, PyPDF2→pypdf); Docker daemon autostart
- `components/clamav-cicap.md` — Added `StructuredSSNFormatStripped no`; handbook `/squid_clamav` gotcha; YARA persistence; scope limitations
- `components/suricata.md` — Added `sysctl dev.netmap.admode=2` failure; Divert `pfctl` diagnostic; XFF/SID 2031071 visibility; Dell PowerEdge IPS readiness
- `components/ioc2rpz.md` — Added server ACL field; GitHub issue #1373; `secondary/` vs `slave/`; `rndc -p 953` in NOTIFY section
- `components/gns3.md` — Added second SNI entry `netbird.sase.local`; mobile01 cabling; overlay roles column; `Fase3-Security-Complete` snapshot
- `overview/architecture.md` — Added SWG identity gap to trust boundaries; CASB Phase 4 planned to component map
- `runbooks/02-ztna-overlay.md` — Added `management.json` validation field names
- `runbooks/03-proxy-wpad.md` — Added `.banking.example.com` to no-bump list
- `runbooks/06-dns-threat-intel.md` — Added ioc2rpz server ACL field
- `runbooks/07-access-policy.md` — Added GeoIP limitation; ~60-70% coverage estimate; multi-OS AV paths; C2 beaconing threat; process_check spoofability
- `findings/iptables-forward-ordering.md` — Added iptables non-persistence warning + `netfilter-persistent` solution
- `findings/ioc2rpz-gui-js-bug.md` — Added permanent fix options (forked image, volume-mounted JS)

**Total files updated:** 22

---

## 2026-04-17 — New source document ingest: SASE_PoC_Testrapport.md

**New source document:** `raw/SASE_PoC_Testrapport.md` — comprehensive F1–F15 acceptance test report (v1.0, April 2026), covering all sandbox test results with exact commands, actual output, and rationale.

**Files created:**

- `testing/acceptance-tests.md` — F1–F15 overall status table, per-test rationale and key output, additional T-A1–T-A9 tests, per-pillar coverage summary, F15 step-by-step breakdown, test environment table
- `decisions/sdwan-descoped.md` — SD-WAN features F12/F13/F14 explicitly descoped (IPsec, QoS, uCPE); Zero Trust rationale; Zscaler alignment; consequence for sitepc01
- `findings/curl-ssl-no-revoke.md` — Windows curl Schannel CRL check fails on SASE-PoC-CA; `--ssl-no-revoke` required for all proxy HTTPS tests from Windows
- `findings/suricata-connection-pooling.md` — Squid connection pooling causes one Suricata alert per SID per flow; not suppression; expected behavior

**Files updated:**

- `index.md` — Added Testing section (1 entry), new decision (SD-WAN descoped), two new findings

**Source document ingested:** `SASE_PoC_Testrapport.md` (1 test report)

---

## 2026-06-03 — Major update: V28–V44 source ingest, new components, event-driven enforcement

**Problem addressed:** The wiki was based on V18–V27 and Doc1–Doc7. Since then, 17 new verslagen (V28–V44) and 10 new implementation documents were produced, introducing: Identity Bridge, NATS JetStream event bus, Control Daemon, Wazuh SIEM, Zitadel Actions, Zero Trust Branch model, CASB three-layer architecture, and full Gate 1+2 implementation. The wiki needed a comprehensive update to reflect the current operational state.

**Scope changes applied:**
- BYOD → managed Windows devices (Intune-managed, Entra joined) per lector mandate R11 (April 21 2026)
- SD-WAN descoped → Zero Trust Branch model with VyOS SASE Gateway
- Gates 1+2 changed from planned to operational (5 CA policies, Intune compliance)
- CASB expanded from partial inline to three-layer model (inline + API + real-time)

**Files created (42 new pages):**

### Components (10 files)
- `components/identity-bridge.md` + `.nl.md` — FastAPI overlay-IP → Entra ID persona group mapping
- `components/nats-jetstream.md` + `.nl.md` — Central event bus connecting detection silos
- `components/control-daemon.md` + `.nl.md` — Threat scoring + quarantine via NetBird API
- `components/wazuh.md` + `.nl.md` — SIEM with NATS forwarder + M365 Active Response
- `components/zitadel.md` + `.nl.md` — OIDC IdP broker with two Zitadel Actions

### Concepts (2 files)
- `concepts/identity-flow.md` + `.nl.md` — Full identity chain: Entra ID → Zitadel → NetBird → Identity Bridge → Squid

### Decisions (14 files)
- `decisions/netbird-service-pat.md` + `.nl.md` — Service-user PAT for API auth
- `decisions/nats-accounts-auth.md` + `.nl.md` — `accounts{}` model for JetStream
- `decisions/groupsync-pad-b.md` + `.nl.md` — Zitadel strips tenant prefix
- `decisions/casb-three-layers.md` + `.nl.md` — Three-layer CASB architecture
- `decisions/managed-devices-scope.md` + `.nl.md` — BYOD → managed devices scope change
- `decisions/zt-sdwan-branch.md` + `.nl.md` — Zero Trust Branch replaces IPsec SD-WAN
- `decisions/control-daemon-scope.md` + `.nl.md` — Threat scoring scope decisions

### Findings (16 files)
- `findings/netbird-issue-3127.md` + `.nl.md`
- `findings/squid-overlay-bind-race.md` + `.nl.md`
- `findings/overlay-ip-instability.md` + `.nl.md`
- `findings/nats-store-dir.md` + `.nl.md`
- `findings/wazuh-cpu-glibc.md` + `.nl.md`
- `findings/wazuh-dashboard-airgate.md` + `.nl.md`
- `findings/netbird-jwt-allow-groups-lockout.md` + `.nl.md`
- `findings/dc-lan-isolation-route-acl.md` + `.nl.md`

### Testing (2 files)
- `testing/attack-scenarios.md` + `.nl.md` — 22 demo validation scenarios by SASE pillar

### Runbooks (8 files)
- `runbooks/08-groupsync.md` + `.nl.md` — JWT group sync, Entra ID token config, Zitadel Actions
- `runbooks/09-identity-bridge.md` + `.nl.md` — FastAPI deployment, Squid external_acl
- `runbooks/10-nats-jetstream.md` + `.nl.md` — Event bus, producers, Control Daemon, Redis
- `runbooks/11-wazuh.md` + `.nl.md` — SIEM stack, NATS forwarder, M365 Active Response

**Files updated (30+ files):**

### Architecture and overview
- `overview/architecture.md` + `.nl.md` — Scope change, new components in mgmt01 plane, identity step in traffic flow, Gate 1+2 operational, new §9 (Event-driven enforcement layer), new §10 (Gate model)
- `index.md` + `.nl.md` — Full catalog update with all 42 new pages, gate status table updated

### Concept pages
- `concepts/sase.md` + `.nl.md` — BYOD→managed, CASB three-layer model with status table
- `concepts/zero-trust.md` + `.nl.md` — Gate 2 now Intune, all gates operational

### Existing component pages (D1-D6)
- `components/squid.md` + `.nl.md` — NATS integration section + Identity Bridge cross-links
- `components/suricata.md` + `.nl.md` — NATS integration section (security.alert.ids)
- `components/clamav-cicap.md` + `.nl.md` — NATS integration section (security.alert.malware)
- `components/python-dlp.md` + `.nl.md` — NATS integration section (security.alert.dlp)
- `components/ioc2rpz.md` + `.nl.md` — NATS integration section (security.alert.dns)
- `components/netbird.md` + `.nl.md` — NATS integration + JWT group sync + app registration update
- `components/vyos.md` + `.nl.md` — Full page replacement (stub → complete Zero Trust Branch)

### Decision pages
- `decisions/sdwan-descoped.md` + `.nl.md` — H1 updated, ZT-Branch section added
- `decisions/ca-posture-hybrid.md` + `.nl.md` — Status → implemented, 5 policies documented
- `decisions/zitadel-idp-broker.md` + `.nl.md` — Zitadel Actions implementation detail added

### Other updates
- `testing/acceptance-tests.md` + `.nl.md` — F3 partly proven, F10+F11 operational, T-A10–T-A13 added
- `runbooks/index.md` + `.nl.md` — 4 new runbooks added, dependency graph updated
- `tags.md` + `.nl.md` — 18 new tags added (6 component, 7 protocol/integration, 5 security concept)
- `log.md` + `.nl.md` — This entry

**Source documents ingested:** V28–V44 (17 verslagen), 10 implementation documents (Addenda E, G, GroupSync, H, I, J, CASB Architecture, ZTSDWAN Architecture, SDWAN Nulmeting, GroupSync Correcties, Correctienotitie CASB, Projectoverzicht Mei 2026, Sessieplanning Juni 2026)

**Total new pages:** 42. **Total updated pages:** 30+.

---

## 2026-06-04 — Raw → Wiki one-directional consistency audit (full pass)

**Goal:** Bring every wiki page into agreement with the raw sources in one direction only —
`raw/` is the single source of truth; only the wiki was changed. Both languages corrected and
EN↔NL divergence treated as a defect. Scope = correctness only (no editorial trimming, no
renames/moves/slug changes).

**Authority model applied:** verslag-confirmed state = ground truth; a later verslag supersedes an
earlier one; correction notes override their parent plan doc but a verslag still wins; addenda /
architecture docs / Doc1–7 = pre-implementation intent (rationale + gap-fill only). This inverts the
rule the wiki was originally built under (see CLAUDE.md fix below), which is why the largest drift
class was pages that had followed a plan/addendum where a verslag later diverged.

**Domains audited (all 12 + cross-cutting Phase E):** DNS-Threat-Intel · SWG/Proxy · Malware+DLP ·
IDS · Lab/Virtualization · ZTNA-Overlay · Context-Aware/CA · GroupSync · Identity-Bridge/Control-Daemon ·
NATS/JetStream · CASB+Wazuh · ZT-SD-WAN · Phase E (overview/architecture, concepts, index, tags,
section indexes, testing/*).

**Findings by class (~88 distinct findings across 86 logical pages × EN+NL):**
- **factual (~28)** — wrong app-reg GUID (`cebe0d74…`→`11803ee8…`), Identity-Bridge poll target
  (`HTTPS overlay`→`http://management:80`), NATS version (`2.14`→`2.14.1-alpine`), Suricata/DNS
  date drift (April→March 2026), Control-Daemon date (May→June), group-name casing
  (`2ITcsc1A-`→`2ITCSC1A-` across 4 pages), VyOS version (`1.4 rolling`→`rolling 2026.02.16`),
  sitepc01 Site-LAN IP (`172.16.10.50`→`172.16.10.10`), and more.
- **stale-superseded (~13)** — NetBird groups/ACL (SASE-* → persona model + single
  `Personas-to-Core-Services` policy, V34); CA posture (NetBird posture → Intune compliance Gate 2,
  V40); sitepc01 (`no OS` → Tiny11 operational, V43/V44); runbook-07/08/09 group tables.
- **status-drift (~10)** — Wazuh CASB live-revoke (working → detect-only HTTP-204 stub, live pending);
  VyOS QoS+failover (planned/descoped → implemented, V43); GroupSync #5399 Dex gotcha (live → not
  applicable, V30.17); acceptance-tests F4/F8 (see below).
- **inconsistency / EN↔NL divergence (~20)** — dominant NL pattern was dropped content (NL pages
  missing blocks the EN had); plus one self-contradiction (`concepts/identity-flow` mislabelled the
  JWT-vs-IdP-Sync *mechanism* as "Path A/B", which raw reserves for prefix handling).
- **ungrounded-and-wrong (~10)** — `components/identity-bridge` invented `/member`, `children-max=5`,
  `identity.login`/`identity.group_change`; `findings/wazuh-dashboard-airgate` invented an
  `api.wazuh.com` URL + "perpetual spinner"; `-p 953` on `rndc` (×4 DNS pages); `control.quarantine`
  NATS subject.
- **link (~7)** — missing Related-runbook links added while editing. Final link audit across all
  172 files (1,259 internal `.md` links) = **0 broken**.

**Key resolutions (authority model in action):**
- **GroupSync Pad A/B:** Pad B shipped (V31/V34) and is a **fail-closed allowlist that maps the
  Entra display-name STRING → clean NetBird name** (`2ITCSC1A-Studenten`→`Studenten`), unmatched
  groups silently dropped — net effect strips the prefix but is stricter than a blind strip. Entra
  security-group casing is **all-caps `2ITCSC1A-`** (V34/V40 verslagen supersede the correction
  note's lowercase `2ITcsc1A-`). Corrected the original wiki's "GUID-keyed" error AND an intermediate
  "generic prefix-strip" over-correction.
- **Quarantine breaks active flow = TESTED, reframed honestly:** V35.14 dry-run on `docent1` —
  EICAR scored 80/80, peer quarantined **within seconds** (persona groups stripped → deny-by-default),
  connectivity lost, restored on un-quarantine. Presented as tested with timing **"within seconds"**
  (not "milliseconds") and WITHOUT the unsupported "page died mid-load" claim. ENFORCE=false until
  Session 11.
- **CASB demo SID:** `100201` → **`100601`** (`AnonymousLinkCreated`, V39 sandbox base 100600 family).
  The seed's `100500/100501` are the Marnix-team bus rules, not the sandbox CASB rules.
- **Acceptance-tests F4/F8 (Datacenter Access / Firewall Segmentation):** reframed as
  **Validated-then-removed** — kept the ✅ build-time result, added current-state framing that the
  DC-LAN-over-overlay path (`Internal-DC` Network + `Datacenter Access` policy) was removed in the V34
  persona migration and deferred; F8's negative isolation half remains valid. Applied to F4, F8, and
  F15 step 6 + tally, EN+NL.

**CLAUDE.md authority rule fixed:** the project instruction "the implementatiedocument wins over the
verslag for current state" was inverted to the corrected model — **verslag-confirmed state is ground
truth; addenda / implementatiedocumenten / plan-documenten are pre-implementation intent and never
override a verslag for current state.** This is the root cause of the systematic drift this audit
corrected.

**Files changed:** wiki pages across all domains (EN+NL) per the per-domain tables in
`.agent-memory/audit-tracker.md`; `CLAUDE.md` (authority rule); `log.md` + `log.nl.md` (this entry).
No `raw/` files were modified. No renames, moves, slug or frontmatter changes.

---

## 2026-06-05 — BYOD + over-translation cleanup + rigorous re-audit

**Goal:** Fix all remaining stale "BYOD" references in current-state descriptions, correct
over-translated Dutch technical terms (~181 instances), then run a full multi-agent re-audit
of every wiki page against the raw verslagen to verify correctness before deployment.

### Phase 1: BYOD cleanup (40 stale references → 0)
All current-state "BYOD client" / "BYOD persona" descriptions replaced with accurate terms
("overlay client", "managed Windows client", etc.) across 55 EN+NL files.

### Phase 2: Over-translated Dutch terms (~181 fixes)
English technical terms that were unnecessarily translated into Dutch were restored:
`eindknooppunt` → exit node, `beleidsregels` → policies, `handtekeningen` → signatures,
`gebeurtenissen` → events, `drempelwaarde` → threshold, `naamserver` → nameserver,
`zonetransfer` → zone transfer, `verbindingspooling` → connection pooling,
`Luisterpoort` → Listen Port, `Doelbronnen` → Target resources, `Toewijzingen` → Assignments,
`Voorwaarden` → Conditions, `Toegangscontroles` → Access controls,
`Aanmeldingslogboeken` → Sign-in logs, and more.

### Phase 3: Full re-audit (4 domain agents + 1 independent Opus auditor)
Four parallel verification agents covered all 12 domains + cross-cutting pages, checking every
wiki claim against the raw verslagen. An independent auditor (Opus model) then cross-checked
the domain agents' findings.

**Domain agent results (combined 37+ 11 + 12 + 30 checks):**
- DNS+SWG+DLP+IDS: ALL PASS
- ZTNA+CA+GroupSync: ALL PASS (37/37)
- Identity+NATS+CASB+Wazuh: PASS with 3 minor gaps (fixed)
- SD-WAN+Lab+CrossCutting: 4 discrepancies (fixed)

**Fixes applied from audit:**
- `decisions/sdwan-descoped.md` + `.nl.md` — F15 step 8 is validated (not N/A); VyOS is
  SASE Gateway with QoS (not "minimal NAT only"); rationale corrected (QoS was reimplemented)
- `overview/architecture.nl.md` — missing "SWG identity layer" trust boundary row added (EN/NL parity)
- `decisions/nats-accounts-auth.md` + `.nl.md` — `$JS.ACK.>` added alongside `$JS.API.>` for consumers (V32)
- `components/control-daemon.md` + `.nl.md` — explicit scoring weights `malware=80`, `dlp_match=30` + V35 test reference
- `components/nats-jetstream.md` + `.nl.md` — 5 JetStream streams table added (V32.13)
- **Independent auditor false positive (reverted):** auditor flagged `security.alert.ids` → `.ips`
  based on V34's snapshot, but the live nats.conf on the server confirms `.ids` is the current
  subject name — the config was updated after V34. Change was applied then reverted.

**Independent auditor summary: 30 PASS, 1 false positive (reverted).** The flagged NATS subject
discrepancy was based on V34's point-in-time observation; the live server config confirms `.ids`.

**Verification sweep results:**
- Over-translated terms: 0 remaining
- BYOD in current-state: 0 (28 remaining all in historical/evolution framing)
- Secrets leaked: 0
- EN↔NL parity: verified across 5 page pairs
- All IPs, ports, versions, group names consistent cross-page

**Files changed:** 63 wiki pages (EN+NL). No `raw/` files modified.

---

## 2026-06-05 — Wazuh dashboard airgate finding rewrite (new source: Wazuh_DB_Fix.md)

**New source document:** `raw/Wazuh_DB_Fix.md` (v1.0, 2 juni 2026) — versie-herstel runbook voor de Wazuh-stack na een `down -v`-incident + versie-skew. Validated 2 juni 2026.

**Root cause corrected:** The existing finding (`findings/wazuh-dashboard-airgate.md`) had the wrong root cause — it blamed the air-gap CTI-500 (`GET /manager/version/check`) for the "Status: Offline" redirect. The new source reveals the redirect is caused by an **empty manager UUID** in `global.db` (wiped by `docker compose down -v`). The CTI-500 is a separate, cosmetic, non-blocking issue in Wazuh 4.14.5+ (fix #8130).

**Status change:** Finding status updated from "known limitation / workaround" to **resolved** (2 juni 2026), fixed via in-place bump to 4.14.5 GA (no `-v`), preserving the existing UUID.

**Audit:** Independent Opus auditor agent verified all planned changes against `Wazuh_DB_Fix.md` before edits were applied. 8 planned edits: all PASS. 2 additional missed files identified by auditor and added to scope.

**Files updated (10):**
- `findings/wazuh-dashboard-airgate.md` + `.nl.md` — complete rewrite: correct root cause (empty UUID), two structural rules, resolution, updated lessons
- `components/wazuh.md` + `.nl.md` — dashboard status updated to "fully operational"; known-issues entry corrected
- `runbooks/11-wazuh.md` + `.nl.md` — gotcha callout updated (resolved); checklist item corrected (app modules no longer gated)
- `index.md` + `.nl.md` — finding blurb corrected (was "hostname resolution issue", now reflects empty UUID cause)

**Source documents ingested:** `Wazuh_DB_Fix.md` (1 versie-herstel runbook). No `raw/` files modified.

---

## 2026-06-05 — NL style corrections (phases 1 + 2)

**Phase 1 — Calques, compounds, sentence structure (34 NL files):**
Dutch calque fixes across all `.nl.md` files: verb calques (bevragen→queryen, oplossen→resolven in DNS context, doorsturen→forwarden), forced compounds (uitgangsknoop→exit node, beveiligingsstapel→security stack), English participium-constructions removed, passive→active rewrites. Based on empirical NL style guide derived from user's actual technical writing style.

**Phase 2 — Em dash removal (82 NL files, ~900 instances):**
All em dashes (—) in running text replaced with contextually appropriate Dutch punctuation: parentheses (preferred for interjections), colons, periods, commas, or semicolons. En dashes in numeric ranges (F1–F15) preserved. Em dashes inside code blocks preserved. Empty table-cell placeholders normalized from `—` to `n.v.t.`. `attack-scenarios.nl.md` double-hyphen (`--`) patterns also fixed for consistency.

**Audit:** Independent Opus auditor agents verified both phases before edits were applied. Phase 1: 10/10 PASS. Phase 2: 18/18 sampled files PASS, 1 pre-existing issue (attack-scenarios.nl.md) caught and fixed.

**Files changed:** 82 NL wiki pages. No EN pages or `raw/` files modified.

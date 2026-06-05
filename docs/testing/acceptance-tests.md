---
title: "Acceptance Test Results (F1–F15)"
tags: [sase, ztna, swg, fwaas, casb, sd-wan, testing]
---

# Acceptance Test Results (F1–F15)

**Version:** 1.0 — April 2026  
**Scope:** Full sandbox test coverage — all F1–F15 acceptance tests (Handboek v4 §46) plus additional validation tests outside the original framework. Pages are organised by rubric criterion with evidence tables mapping each test to what it proves.  
**Source:** `SASE_PoC_Testrapport.md`

---

## Overall status

| Test | Name | Pillar | Status |
|------|------|--------|--------|
| **F1** | ZTNA Tunnel Connectivity | ZTNA | ✅ Validated |
| **F2** | Entra ID SSO | ZTNA | ✅ Validated |
| **F3** | Posture Check | ZTNA | ⏳ Partly proven — CA policies active, Intune enrolled (Report-only) |
| **F4** | Datacenter Access via ZTNA | ZTNA | ✅ Validated (build-time, May 2026) — DC-LAN-over-overlay path since removed in V34, deferred |
| **F5** | URL Filtering | SWG | ✅ Validated |
| **F6** | SSL Bump Inspection | SWG | ✅ Validated |
| **F7** | Malware Detection (ClamAV) | SWG | ✅ Validated |
| **F8** | Firewall Segmentation | ZTNA | ✅ Validated — DC-LAN isolation holds (unenrolled = no route); positive ACL path removed in V34 |
| **F9** | Suricata Alert Generation | General | ✅ Validated (extended beyond original definition) |
| **F10** | Central Log Aggregation (SIEM) | General | ✅ Operational — Wazuh + NATS forwarder + pop01 agent |
| **F11** | CASB Alert and Remediation | CASB | ✅ Functional — Wazuh + M365 Management Activity API + Active Response |
| **F12** | IPsec Tunnel Connectivity | SD-WAN | ✖ N/A — architectural decision (see [SD-WAN Descoped](../decisions/sdwan-descoped.md)) |
| **F13** | QoS Traffic Classification | SD-WAN | ✖ N/A as a classic-IPsec test — QoS is instead implemented + validated under the ZT-Branch model (V43 Test #5; see [ZT SD-WAN Branch](../decisions/zt-sdwan-branch.md)) |
| **F14** | Datacenter Access via SD-WAN | SD-WAN | ✖ N/A — architectural decision |
| **F15** | Full-Stack SASE Validation | Integration | ✅ Partial — steps 1–6 + 8 validated, step 7 N/A (classic IPsec SD-WAN), step 9 operational (SIEM via Wazuh) |

### Additional validated tests (outside F1–F15)

| Test | Name | Status |
|------|------|--------|
| **T-A1** | DLP YARA — CONFIDENTIAL label detection in download | ✅ Validated |
| **T-A2** | DLP SDD — StructuredDataDetection threshold (4× CC → block, 1× CC → pass) | ✅ Validated |
| **T-A3** | DLP ICAP REQMOD — Python DLP upload blocking (Luhn CC in POST) | ✅ Validated |
| **T-A4** | DNS RPZ — NXDOMAIN + aa-flag from pop01 local | ✅ Validated |
| **T-A5** | DNS RPZ — NXDOMAIN from mobile01 via NetBird overlay | ✅ Validated |
| **T-A6** | DNS RPZ — NXDOMAIN from dc01 via DC-LAN | ✅ Validated |
| **T-A7** | Suricata vtnet1 (LAN) — DC-LAN traffic detection | ✅ Validated |
| **T-A8** | Suricata suspicious User-Agent | ✅ Validated |
| **T-A9** | Suricata DNS anomaly detection | ✅ Validated |
| **T-A10** | Identity Bridge — overlay IP to persona group resolution | ✅ Validated |
| **T-A11** | NATS event bus — cross-component event delivery | ✅ Validated |
| **T-A12** | Control Daemon — threat score accumulation + quarantine | ✅ Validated |
| **T-A13** | Wazuh SIEM — NATS event ingestion + dashboard query | ✅ Validated |

---

## Coverage per SASE pillar

| Pillar | Validated | Planned | N/A |
|--------|-----------|---------|-----|
| **SWG** | F5, F6, F7, T-A1, T-A2, T-A3, T-A4/A5/A6 | — | — |
| **CASB** | F11, T-A10, T-A11, T-A12, T-A13 | — | — |
| **ZTNA** | F1, F2, F4, F8 | F3 (architecture ready) | — |
| **SD-WAN** | QoS + failover detection (ZT-Branch, V43 Test #5/#6) | — | F12, F13, F14 (classic IPsec) |
| **General / Integration** | F9, F10, F15 | — | — |

---

## 1. SWG (Secure Web Gateway)

### 1.1 Traffic routing (no local breakout)

**Rubric excellent:** Fully forced through SWG

| Test | What it proves |
|------|----------------|
| F1 | `InterfaceAlias: wt0` proves all traffic exits via the WireGuard tunnel |
| F5 | `curl -x http://100.70.154.79:3128` proves traffic flows through Squid (SWG) |
| PAC file | `FindProxyForURL` returns `PROXY 100.70.154.79:3128` for all external traffic |
| Wiki evidence | Runbook 03 (WPAD/PAC configuration), Decision: WPAD vs transparent proxy |

### 1.2 Identity-based access

**Rubric excellent:** Fully integrated with Microsoft Entra ID

| Test | What it proves |
|------|----------------|
| T-A10 | Identity Bridge resolves overlay IP to Entra ID persona group |
| F2 | OIDC chain: NetBird to Zitadel to Entra ID to JWT with group claims |
| Wiki evidence | Component: Identity Bridge, Concept: Identity Flow, Runbook 09 |

### 1.3 DNS filtering

**Rubric excellent:** Advanced (threat intel, logging, policy-based)

| Test | What it proves |
|------|----------------|
| T-A4 | RPZ NXDOMAIN + aa-flag locally on pop01 |
| T-A5 | RPZ NXDOMAIN via overlay (mobile01) |
| T-A6 | RPZ NXDOMAIN via DC-LAN (dc01) |
| Wiki evidence | Component: ioc2rpz (71,767 records, URLhaus + ThreatFox feeds), Concept: RPZ |

### 1.4 Malware inspection (DPI)

**Rubric excellent:** DPI + TLS decryption + malware detection

| Test | What it proves |
|------|----------------|
| F6 | SSL Bump active: cert issuer = "SASE PoC" (TLS decryption) |
| F7 | EICAR blocked via ClamAV/c-icap RESPMOD (malware detection) |
| T-A1 | YARA DLP: CONFIDENTIAL label detection on download |
| T-A2 | SDD: Luhn-validated credit card detection with threshold |
| T-A3 | Python DLP REQMOD: upload blocking (DPI on uploads) |
| Wiki evidence | Component: ClamAV/c-icap, Component: Python DLP, Concept: DLP, Decision: two-layer DLP |

### 1.5 Configuration and documentation

**Rubric excellent:** Highly detailed

| Evidence | What it proves |
|----------|----------------|
| Wiki itself | Component pages with configuration, runbooks with step-by-step instructions |
| Handboek v4 | 5,350-line implementation handbook |
| Decisions section | 17 ADRs with context, options, and consequences |
| Findings section | 24 documented gotchas with root cause and resolution |

### 1.6 Testing and validation

**Rubric excellent:** Includes malware test and DNS blocking validation

| Test | What it proves |
|------|----------------|
| F7 | EICAR malware test end-to-end |
| T-A4/5/6 | DNS blocking validation across three segments |
| Attack scenarios | Bypass attempts per pillar |
| Wiki evidence | Testing: attack-scenarios, Testing: demo-script |

---

## 2. CASB (Cloud Access Security Broker)

### 2.1 Blocking cloud apps

**Rubric excellent:** Dynamic and context-based blocking

| Test | What it proves |
|------|----------------|
| F5 | URL filtering blocks specific sites (gambling.com, etc.) |
| F11 | CASB alert: SharePoint SharingSet/AnonymousLinkCreated detection |
| T-A12 | Control Daemon quarantine: peer removed from persona group results in deny-by-default |
| Wiki evidence | Decision: CASB three layers, Component: Wazuh (Layer 2), Component: Control Daemon (Layer 3) |

### 2.2 Identity integration

**Rubric excellent:** Fully integrated with Microsoft Entra ID

| Test | What it proves |
|------|----------------|
| T-A10 | Identity Bridge: overlay IP to Entra ID persona group |
| T-A11 | NATS event bus: identity events flow cross-component |
| F2 | Entra ID SSO via OIDC chain |
| Wiki evidence | Concept: Identity Flow, Component: Identity Bridge, Runbook 08 (GroupSync) |

### 2.3 Policy configuration

**Rubric excellent:** Advanced policies (groups, risk, device)

| Test | What it proves |
|------|----------------|
| F3 | CA policies: MFA, geo-block, legacy-auth, risk-based, compliant device |
| T-A10 | Persona groups (Studenten/Docenten/Admins) drive policy |
| Wiki evidence | Runbook 07 (5 CA policies), Decision: CA + Posture hybrid |

### 2.4 Test results

**Rubric excellent:** Extensive test cases and bypass attempts

| Test | What it proves |
|------|----------------|
| T-A12 | Control Daemon: EICAR score 80/80, quarantine, recovery |
| T-A13 | Wazuh SIEM: NATS event ingestion and dashboard query |
| Attack scenarios | C1–C5: CASB-specific bypass scenarios |
| Wiki evidence | Testing: attack-scenarios (CASB section), Testing: demo-script |

### 2.5 Limitations and analysis

**Rubric excellent:** Critical reflection and improvement proposals

| Evidence | What it proves |
|----------|----------------|
| Decision: CASB three layers | Honest gap analysis: "thinner dimensions" explicitly named |
| Decision: Control Daemon scope | proxy_block false positives removed, IDS correlation deliberately not implemented |
| Findings | 24 findings with root cause analysis |
| Wiki evidence | "Possible improvements" section in casb-three-layers |

---

## 3. ZTNA (Zero Trust Network Access)

### 3.1 Per-application access

**Rubric excellent:** Full zero trust model (no network exposure)

| Test | What it proves |
|------|----------------|
| F4 | DC-LAN reachable only via NetBird Networks and group membership |
| F8 | Unenrolled device: no route to 10.0.0.0/24 |
| Wiki evidence | Component: NetBird (ACL policies, Networks vs Routes), Concept: Zero Trust |

### 3.2 Identity and context awareness

**Rubric excellent:** Identity + context (geoIP, device posture, malware check)

| Test | What it proves |
|------|----------------|
| F2 | Identity: Entra ID SSO, peer appears with Entra ID username |
| F3 | Context: 5 CA policies (MFA, geo-block, legacy-auth, risk-based, compliant device) |
| F3 | Device posture: Intune compliance (OS version, Defender AV, firewall) |
| Wiki evidence | Decision: CA + Posture hybrid (three-gate model), Runbook 07 |

### 3.3 Per-application tunnel

**Rubric excellent:** Dynamic per-app tunnels correctly established

| Test | What it proves |
|------|----------------|
| F1 | WireGuard tunnel operational: `InterfaceAlias: wt0`, peer count 1/1 Connected |
| F8 | ACL policy per resource: Datacenter Access requires explicit group membership |
| Wiki evidence | Component: NetBird (ACL policies per resource group) |

### 3.4 Access to on-premises resources

**Rubric excellent:** Secure, conditional, and logged

| Test | What it proves |
|------|----------------|
| F4 | DC-LAN reachable via wt0 (WireGuard-encrypted) |
| F3 | Conditional: CA policies evaluated at every login |
| T-A13 | Logged: Wazuh SIEM receives and indexes events |
| Wiki evidence | Component: Wazuh, Runbook 11 |

### 3.5 Testing and validation

**Rubric excellent:** Attacks simulated and logging analysed

| Test | What it proves |
|------|----------------|
| F8 | Negative test: unenrolled device cannot reach DC-LAN |
| Attack scenarios | A1–A3: ZTNA-specific attack scenarios |
| T-A13 | Wazuh logging: events queryable in Discover dashboard |
| Wiki evidence | Testing: attack-scenarios (ZTNA section) |

---

## 4. SD-WAN

### 4.1 Basic connectivity

**Rubric excellent:** Fully operational with failover

| Test | What it proves |
|------|----------------|
| V43 Test #6 | Failover detection: CRITICAL within 30 s on interface-down, recovery logged |
| sitepc01 enrollment | NetBird-enrolled via Entra ID, operational on Site-LAN |
| Wiki evidence | Component: VyOS, Decision: ZT SD-WAN Branch |

### 4.2 Routing and policies

**Rubric excellent:** Smart routing (latency, failover)

| Test | What it proves |
|------|----------------|
| V43 Test #5 | QoS: DSCP EF 300/300 packets, 0 drops; bulk: 26 drops and 17k overlimits |
| VyOS tc shaper | `tc` policy on eth0 with EF/default classes |
| Wiki evidence | Component: VyOS (QoS configuration), Decision: ZT SD-WAN Branch |

### 4.3 Integration with SASE

**Rubric excellent:** Clear role within SASE

| Evidence | What it proves |
|----------|----------------|
| Decision: ZT SD-WAN Branch | VyOS as SASE gateway, aligned with Zscaler ZT-SD-WAN model |
| Decision: SD-WAN descoped | Classic IPsec rejected on architectural grounds (Zero Trust) |
| Wiki evidence | Architecture page, Concept: SASE |

### 4.4 Configuration

**Rubric excellent:** Well documented

| Evidence | What it proves |
|----------|----------------|
| Component: VyOS | Full configuration: QoS shaper, health check, NAT |
| Wiki evidence | Decision: ZT SD-WAN Branch with test results |

### 4.5 Testing

**Rubric excellent:** Multiple scenarios tested

| Test | What it proves |
|------|----------------|
| V43 Test #5 | QoS under load |
| V43 Test #6 | Failover detection |
| F12/13/14 | N/A as classic IPsec, replaced by ZT-Branch tests |
| Wiki evidence | Decision: ZT SD-WAN Branch (evidence section) |

---

## 5. General

### 5.1 Architecture

**Rubric excellent:** Professional

Wiki evidence: Architecture page, control/data plane split, node roles.

### 5.2 Functional diagram (packet flow)

**Rubric excellent:** Complete SASE flow

Wiki evidence: Interactive functional diagram (`demos/functioneel-schema.html`).

### 5.3 Packet flow explanation

**Rubric excellent:** Step-by-step analysis

Wiki evidence: Architecture §5.1–5.4 (four traffic flows written out).

### 5.4 Transparency for the customer (logging and insight)

**Rubric excellent:** Clear explanation of why traffic is blocked or allowed

Wiki evidence: F5 (Squid ERR_ACCESS_DENIED), F7 (ClamAV VIRUS DETECTED log), T-A4/5/6 (RPZ NXDOMAIN + aa-flag), Component: Wazuh (SIEM dashboard), F10.

### 5.5 Performance optimisation

**Rubric excellent:** Actively optimised (routing, caching, tuning)

Wiki evidence: Suricata Hyperscan engine, Squid `dynamic_cert_mem_cache`, VyOS QoS shaper tuning.

### 5.6 Choice of open-source tools

**Rubric excellent:** Strongly justified

Wiki evidence: 17 decision pages (ADR format), Concept: SASE (commercial equivalents table).

### 5.7 Component integration

**Rubric excellent:** Fully integrated

Wiki evidence: T-A11 (NATS cross-component event delivery), T-A12 (Control Daemon end-to-end), NATS event bus architecture, Identity Bridge SWG+ZTNA coupling.

### 5.8 Troubleshooting

**Rubric excellent:** In-depth analysis

Wiki evidence: 24 findings pages with root cause, diagnosis, resolution, and verification.

### 5.9 Reporting

**Rubric excellent:** Professional

Wiki evidence: Wiki itself, Handboek v4, Doc1–Doc7, Addenda A–J, 44 session reports.

### 5.10 Collaboration (teamwork)

**Rubric excellent:** Proactive collaboration, knowledge sharing, and joint problem solving

Wiki evidence: Sandbox as reference implementation for the team, Doc1–Doc7 as comparison documents, wiki as shared knowledge base.

---

## Test environment

| Aspect | Value |
|--------|-------|
| Snapshot at validation | `Fase2-ZTNA-Complete` (base), then Phase 3 incremental |
| pop01 RAM | 8 GB (increased from 4 GB for concurrent Suricata + ClamAV load) |
| ClamAV signatures | daily v27953, main v63, bytecode v339 — 3,642,437 signatures |
| Suricata rules | ET Open + Abuse.ch: **79,620+** rules (Hyperscan engine) |
| RPZ records | URLhaus + ThreatFox: **71,767** records |
| mobile01 | Windows 11, VMware, external — simulates remote managed Windows client outside school network |
| NetBird overlay range | 100.64.0.0/10 |

---

## Related

- [Architecture](../overview/architecture.md)
- [Decision: SD-WAN Descoped](../decisions/sdwan-descoped.md)
- [Decision: CA + Posture hybrid (Three-Gate Model)](../decisions/ca-posture-hybrid.md)
- [Finding: curl --ssl-no-revoke on Windows](../findings/curl-ssl-no-revoke.md)
- [Finding: Suricata connection pooling](../findings/suricata-connection-pooling.md)
- [NetBird](../components/netbird.md)
- [Squid](../components/squid.md)
- [ClamAV/c-icap](../components/clamav-cicap.md)
- [Python DLP](../components/python-dlp.md)
- [Suricata](../components/suricata.md)
- [ioc2rpz](../components/ioc2rpz.md)

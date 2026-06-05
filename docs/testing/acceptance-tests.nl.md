---
title: "Acceptatietestresultaten (F1–F15)"
tags: [sase, ztna, swg, fwaas, casb, sd-wan, testing]
---

# Acceptatietestresultaten (F1–F15)

**Versie:** 1.0, april 2026  
**Scope:** Volledige sandboxtestdekking: alle F1–F15-acceptatietests (Handboek v4 §46) plus aanvullende validatietests buiten het oorspronkelijke kader.  
**Bron:** `SASE_PoC_Testrapport.md`

---

## Algemene status

| Test | Naam | Pijler | Status |
|------|------|--------|--------|
| **F1** | ZTNA-tunnelverbinding | ZTNA | ✅ Gevalideerd |
| **F2** | Entra ID SSO | ZTNA | ✅ Gevalideerd |
| **F3** | Posturecontrole | ZTNA | ⏳ Deels bewezen; CA-beleid actief, Intune-enrolled (Report-only) |
| **F4** | Datacentertoegang via ZTNA | ZTNA | ✅ Gevalideerd (bij opbouw, mei 2026); DC-LAN-over-overlay-pad sindsdien verwijderd in V34, uitgesteld |
| **F5** | URL-filtering | SWG | ✅ Gevalideerd |
| **F6** | SSL-Bump-inspectie | SWG | ✅ Gevalideerd |
| **F7** | Malwaredetectie (ClamAV) | SWG | ✅ Gevalideerd |
| **F8** | Firewall-segmentatie | ZTNA | ✅ Gevalideerd; DC-LAN-isolatie blijft gelden (niet-ge-enrolld = geen route); positief ACL-pad verwijderd in V34 |
| **F9** | Suricata-alertgeneratie | Algemeen | ✅ Gevalideerd (uitgebreid voorbij oorspronkelijke definitie) |
| **F10** | Centrale logaggregatie (SIEM) | Algemeen | ✅ Operationeel; Wazuh + NATS-forwarder + pop01-agent |
| **F11** | CASB-alert en herstel | CASB | ✅ Functioneel; Wazuh + M365 Management Activity API + Active Response |
| **F12** | IPsec-tunnelverbinding | SD-WAN | ✖ N.v.t., architectuurbeslissing (zie [SD-WAN Geschrapt](../decisions/sdwan-descoped.md)) |
| **F13** | QoS-verkeersclassificatie | SD-WAN | ✖ N.v.t. als klassieke-IPsec-test; QoS is in plaats daarvan geïmplementeerd + gevalideerd onder het ZT-Branch-model (V43 Test #5; zie [ZT SD-WAN Branch](../decisions/zt-sdwan-branch.md)) |
| **F14** | Datacentertoegang via SD-WAN | SD-WAN | ✖ N.v.t., architectuurbeslissing |
| **F15** | Volledige SASE-validatie | Integratie | ✅ Gedeeltelijk; stappen 1–6 + 8 gevalideerd, stap 7 N.v.t. (klassiek IPsec SD-WAN), stap 9 operationeel (SIEM via Wazuh) |

### Aanvullende gevalideerde tests (buiten F1–F15)

| Test | Naam | Status |
|------|------|--------|
| **T-A1** | DLP YARA: detectie van CONFIDENTIAL-label bij download | ✅ Gevalideerd |
| **T-A2** | DLP SDD: StructuredDataDetection-threshold (4× CC → blokkeren, 1× CC → doorlaten) | ✅ Gevalideerd |
| **T-A3** | DLP ICAP REQMOD: Python DLP uploadblokkering (Luhn CC in POST) | ✅ Gevalideerd |
| **T-A4** | DNS RPZ: NXDOMAIN + aa-vlag van pop01 lokaal | ✅ Gevalideerd |
| **T-A5** | DNS RPZ: NXDOMAIN van mobile01 via NetBird-overlay | ✅ Gevalideerd |
| **T-A6** | DNS RPZ: NXDOMAIN van dc01 via DC-LAN | ✅ Gevalideerd |
| **T-A7** | Suricata vtnet1 (LAN): DC-LAN-verkeersdetectie | ✅ Gevalideerd |
| **T-A8** | Suricata verdachte User-Agent | ✅ Gevalideerd |
| **T-A9** | Suricata DNS-anomaliedetectie | ✅ Gevalideerd |
| **T-A10** | Identity Bridge: overlay-IP naar personagroepresolutie | ✅ Gevalideerd |
| **T-A11** | NATS-eventbus: cross-component event delivery | ✅ Gevalideerd |
| **T-A12** | Control Daemon: dreigingsscoreaccumulatie + quarantaine | ✅ Gevalideerd |
| **T-A13** | Wazuh SIEM: NATS event ingestion + dashboardquery | ✅ Gevalideerd |

---

## Dekking per SASE-pijler

| Pijler | Gevalideerd | Gepland | N.v.t. |
|--------|-------------|---------|--------|
| **SWG** | F5, F6, F7, T-A1, T-A2, T-A3, T-A4/A5/A6 | n.v.t. | n.v.t. |
| **CASB** | F11, T-A10, T-A11, T-A12, T-A13 | n.v.t. | n.v.t. |
| **ZTNA** | F1, F2, F4, F8 | F3 (architectuur gereed) | n.v.t. |
| **SD-WAN** | QoS + failover-detectie (ZT-Branch, V43 Test #5/#6) | n.v.t. | F12, F13, F14 (klassiek IPsec) |
| **Algemeen / Integratie** | F9, F10, F15 | n.v.t. | n.v.t. |

---

## 1. SWG (Secure Web Gateway)

### 1.1 Traffic routing (geen local breakout)

**Rubric criterium:** Volledig geforceerd via SWG

| Test | Wat het bewijst |
|------|----------------|
| F1 | `InterfaceAlias: wt0` bewijst dat al het verkeer via de WireGuard-tunnel vertrekt |
| F5 | `curl -x http://100.70.154.79:3128` bewijst dat verkeer via Squid (SWG) loopt |
| PAC-bestand | `FindProxyForURL` retourneert `PROXY 100.70.154.79:3128` voor al het externe verkeer |
| Wiki-bewijs | Runbook 03 (WPAD/PAC-configuratie), Beslissing: WPAD vs transparante proxy |

### 1.2 Identity-based access

**Rubric criterium:** Volledig geïntegreerd met Microsoft Entra ID

| Test | Wat het bewijst |
|------|----------------|
| T-A10 | Identity Bridge resolvet overlay-IP naar Entra ID persona-groep |
| F2 | OIDC-keten: NetBird naar Zitadel naar Entra ID naar JWT met groepsclaims |
| Wiki-bewijs | Component: Identity Bridge, Concept: Identity Flow, Runbook 09 |

### 1.3 DNS-filtering

**Rubric criterium:** Geavanceerd (threat intel, logging, policy-based)

| Test | Wat het bewijst |
|------|----------------|
| T-A4 | RPZ NXDOMAIN + aa-vlag lokaal op pop01 |
| T-A5 | RPZ NXDOMAIN via overlay (mobile01) |
| T-A6 | RPZ NXDOMAIN via DC-LAN (dc01) |
| Wiki-bewijs | Component: ioc2rpz (71.767 records, URLhaus + ThreatFox feeds), Concept: RPZ |

### 1.4 Malware-inspectie (DPI)

**Rubric criterium:** DPI + TLS-decryptie + malware-detectie

| Test | Wat het bewijst |
|------|----------------|
| F6 | SSL Bump actief: cert issuer = "SASE PoC" (TLS-decryptie) |
| F7 | EICAR geblokkeerd via ClamAV/c-icap RESPMOD (malware-detectie) |
| T-A1 | YARA DLP: CONFIDENTIAL-label detectie bij download |
| T-A2 | SDD: Luhn-gevalideerde creditcarddetectie met threshold |
| T-A3 | Python DLP REQMOD: upload-blokkering (DPI op uploads) |
| Wiki-bewijs | Component: ClamAV/c-icap, Component: Python DLP, Concept: DLP, Beslissing: twee-laags DLP |

### 1.5 Configuratie en documentatie

**Rubric criterium:** Zeer gedetailleerd

| Bewijs | Wat het bewijst |
|--------|----------------|
| Wiki zelf | Componentpagina's met configuratie, runbooks met stapsgewijze instructies |
| Handboek v4 | 5350 regels implementatiehandboek |
| Beslissingen-sectie | 17 ADR's met context, opties en gevolgen |
| Bevindingen-sectie | 24 gedocumenteerde gotchas met root cause en oplossing |

### 1.6 Test en validatie

**Rubric criterium:** Inclusief malware-test en DNS-blockingvalidatie

| Test | Wat het bewijst |
|------|----------------|
| F7 | EICAR malware-test end-to-end |
| T-A4/5/6 | DNS-blockingvalidatie over drie segmenten |
| Aanvalsscenario's | Bypass-pogingen per pijler |
| Wiki-bewijs | Testing: attack-scenarios, Testing: demo-script |

---

## 2. CASB (Cloud Access Security Broker)

### 2.1 Blokkeren van cloud apps

**Rubric criterium:** Dynamische en context-based blokkering

| Test | Wat het bewijst |
|------|----------------|
| F5 | URL-filtering blokkeert specifieke sites (gambling.com, etc.) |
| F11 | CASB-alert: SharePoint SharingSet/AnonymousLinkCreated detectie |
| T-A12 | Control Daemon quarantaine: peer uit persona-groep resulteert in deny-by-default |
| Wiki-bewijs | Beslissing: CASB drie lagen, Component: Wazuh (Laag 2), Component: Control Daemon (Laag 3) |

### 2.2 Integratie met identity

**Rubric criterium:** Volledig geïntegreerd met Microsoft Entra ID

| Test | Wat het bewijst |
|------|----------------|
| T-A10 | Identity Bridge: overlay-IP naar Entra ID persona-groep |
| T-A11 | NATS event bus: identity events stromen cross-component |
| F2 | Entra ID SSO via OIDC-keten |
| Wiki-bewijs | Concept: Identity Flow, Component: Identity Bridge, Runbook 08 (GroupSync) |

### 2.3 Policy-configuratie

**Rubric criterium:** Geavanceerde policies (groepen, risico, apparaat)

| Test | Wat het bewijst |
|------|----------------|
| F3 | CA policies: MFA, geo-block, legacy-auth, risk-based, compliant device |
| T-A10 | Persona-groepen (Studenten/Docenten/Admins) sturen policy |
| Wiki-bewijs | Runbook 07 (5 CA policies), Beslissing: CA + Posture hybride |

### 2.4 Testresultaten

**Rubric criterium:** Uitgebreide testcases en bypass-pogingen

| Test | Wat het bewijst |
|------|----------------|
| T-A12 | Control Daemon: EICAR-score 80/80, quarantaine, herstel |
| T-A13 | Wazuh SIEM: NATS-event ingestie en dashboard query |
| Aanvalsscenario's | C1–C5: CASB-specifieke bypass-scenario's |
| Wiki-bewijs | Testing: attack-scenarios (CASB-sectie), Testing: demo-script |

### 2.5 Beperkingen en analyse

**Rubric criterium:** Kritische reflectie en verbetervoorstellen

| Bewijs | Wat het bewijst |
|--------|----------------|
| Beslissing: CASB drie lagen | Eerlijke gap-analyse: "dunnere dimensies" benoemd |
| Beslissing: Control Daemon scope | proxy_block false positives verwijderd, IDS-correlatie bewust niet geïmplementeerd |
| Bevindingen | 24 findings met root cause-analyse |
| Wiki-bewijs | "Mogelijke verbeteringen"-sectie in casb-three-layers |

---

## 3. ZTNA (Zero Trust Network Access)

### 3.1 Per-applicatietoegang

**Rubric criterium:** Volledig zero trust model (geen netwerk exposure)

| Test | Wat het bewijst |
|------|----------------|
| F4 | DC-LAN alleen bereikbaar via NetBird Networks en groepslidmaatschap |
| F8 | Niet-ingeschreven apparaat: geen route naar 10.0.0.0/24 |
| Wiki-bewijs | Component: NetBird (ACL policies, Networks vs Routes), Concept: Zero Trust |

### 3.2 Identity en context awareness

**Rubric criterium:** Identity + context (geoIP, device posture, malware check)

| Test | Wat het bewijst |
|------|----------------|
| F2 | Identity: Entra ID SSO, peer verschijnt met Entra ID-gebruikersnaam |
| F3 | Context: 5 CA policies (MFA, geo-block, legacy-auth, risk-based, compliant device) |
| F3 | Device posture: Intune compliance (OS-versie, Defender AV, firewall) |
| Wiki-bewijs | Beslissing: CA + Posture hybride (drie-gate model), Runbook 07 |

### 3.3 Tunnel per applicatie

**Rubric criterium:** Dynamische per-app tunnels correct opgezet

| Test | Wat het bewijst |
|------|----------------|
| F1 | WireGuard-tunnel operationeel: `InterfaceAlias: wt0`, peer count 1/1 Connected |
| F8 | ACL-beleid per resource: Datacenter Access vereist expliciet groepslidmaatschap |
| Wiki-bewijs | Component: NetBird (ACL policies per resourcegroep) |

### 3.4 Toegang tot on-premises resources

**Rubric criterium:** Veilig, conditioneel en gelogd

| Test | Wat het bewijst |
|------|----------------|
| F4 | DC-LAN bereikbaar via wt0 (WireGuard-encrypted) |
| F3 | Conditioneel: CA policies evalueren bij elke login |
| T-A13 | Gelogd: Wazuh SIEM ontvangt en indexeert events |
| Wiki-bewijs | Component: Wazuh, Runbook 11 |

### 3.5 Test en validatie

**Rubric criterium:** Aanvallen gesimuleerd en logging geanalyseerd

| Test | Wat het bewijst |
|------|----------------|
| F8 | Negatieve test: niet-ingeschreven apparaat kan DC-LAN niet bereiken |
| Aanvalsscenario's | A1–A3: ZTNA-specifieke aanvalsscenario's |
| T-A13 | Wazuh-logging: events opvraagbaar in Discover-dashboard |
| Wiki-bewijs | Testing: attack-scenarios (ZTNA-sectie) |

---

## 4. SD-WAN

### 4.1 Basisconnectiviteit

**Rubric criterium:** Volledig werkend met failover

| Test | Wat het bewijst |
|------|----------------|
| V43 Test #6 | Failover-detectie: CRITICAL binnen 30 sec bij interface-down, herstel gelogd |
| sitepc01 enrollment | NetBird-enrolled via Entra ID, operationeel op Site-LAN |
| Wiki-bewijs | Component: VyOS, Beslissing: ZT SD-WAN Branch |

### 4.2 Routing en policies

**Rubric criterium:** Slimme routing (latency, failover)

Latency en routing worden hier ingevuld door QoS-prioritering en een applicatie-based split-tunnel, niet door latency-gemeten padkeuze. Dat laatste vereist meerdere WAN-paden; het Zero Trust Branch-model heeft er bewust één, met inspectie gecentraliseerd op de PoP.

| Test | Wat het bewijst |
|------|----------------|
| V43 Test #5 | QoS DSCP EF strict-priority: 300/300 pakketten, 0 drops; bulk 26 drops + 17k overlimits. Beschermt realtime-media-latency onder congestie |
| Split-tunnel | Teams Optimize-media verlaat de site rechtstreeks via eth0 (Intune `/14`-route) i.p.v. door de WireGuard-overlay: applicatie-based routing die de overlay-hairpin vermijdt |
| V43 Test #6 | Failover-detectie: CRITICAL binnen 30 sec bij interface-down, herstel gelogd (detectie + alerting, geen automatische dual-WAN-switch) |
| Wiki-bewijs | Component: VyOS (QoS + split-tunnel), Beslissing: ZT SD-WAN Branch |

### 4.3 Integratie met SASE

**Rubric criterium:** Duidelijke rol binnen SASE

| Bewijs | Wat het bewijst |
|--------|----------------|
| Beslissing: ZT SD-WAN Branch | VyOS als SASE-gateway, aligned met Zscaler ZT-SD-WAN model |
| Beslissing: SD-WAN geschrapt | Klassiek IPsec verworpen op architecturele gronden (Zero Trust) |
| Wiki-bewijs | Architectuurpagina, Concept: SASE |

### 4.4 Configuratie

**Rubric criterium:** Goed gedocumenteerd

| Bewijs | Wat het bewijst |
|--------|----------------|
| Component: VyOS | Volledige configuratie: QoS shaper, health check, NAT |
| Wiki-bewijs | Beslissing: ZT SD-WAN Branch met testresultaten |

### 4.5 Test

**Rubric criterium:** Meerdere scenario's getest

| Test | Wat het bewijst |
|------|----------------|
| V43 Test #5 | QoS onder last |
| V43 Test #6 | Failover-detectie |
| F12/13/14 | n.v.t. als klassiek IPsec, vervangen door ZT-Branch tests |
| Wiki-bewijs | Beslissing: ZT SD-WAN Branch (bewijs-sectie) |

---

## 5. Algemeen

### 5.1 Architectuur

**Rubric criterium:** Professioneel

Wiki-bewijs: Architectuurpagina, control/data plane-splitsing, node-rollen.

### 5.2 Functioneel schema (packet flow)

**Rubric criterium:** Volledige SASE flow

Wiki-bewijs: Interactief functioneel schema (`demos/functioneel-schema.html`).

### 5.3 Pakketflow-uitleg

**Rubric criterium:** Stap-voor-stap analyse

Wiki-bewijs: Architectuur §5.1–5.4 (vier verkeersstromen uitgeschreven).

### 5.4 Transparantie voor klant (logging en inzicht)

**Rubric criterium:** Duidelijke uitleg waarom verkeer geblokkeerd of toegestaan is

Wiki-bewijs: F5 (Squid ERR_ACCESS_DENIED), F7 (ClamAV VIRUS DETECTED log), T-A4/5/6 (RPZ NXDOMAIN + aa-vlag), Component: Wazuh (SIEM-dashboard), F10.

### 5.5 Performantie-optimalisatie

**Rubric criterium:** Actief geoptimaliseerd (routing, caching, tuning)

Wiki-bewijs: Suricata Hyperscan engine, Squid `dynamic_cert_mem_cache`, VyOS QoS shaper tuning.

### 5.6 Keuze open-source tools

**Rubric criterium:** Sterk onderbouwd

Wiki-bewijs: 17 beslissingspagina's (ADR-formaat), Concept: SASE (commerciële equivalenten-tabel).

### 5.7 Integratie componenten

**Rubric criterium:** Volledig geïntegreerd

Wiki-bewijs: T-A11 (NATS cross-component event delivery), T-A12 (Control Daemon end-to-end), NATS event bus-architectuur, Identity Bridge SWG+ZTNA-koppeling.

### 5.8 Troubleshooting

**Rubric criterium:** Diepgaande analyse

Wiki-bewijs: 24 bevindingenpagina's met root cause, diagnose, oplossing en verificatie.

### 5.9 Rapportering

**Rubric criterium:** Professioneel

Wiki-bewijs: Wiki zelf, Handboek v4, Doc1–Doc7, Addenda A–J, 44 verslagen.

### 5.10 Samenwerking (teamwork)

**Rubric criterium:** Proactieve samenwerking, kennisdeling en gezamenlijke probleemoplossing

Wiki-bewijs: Sandbox als referentie-implementatie voor team, Doc1–Doc7 als vergelijkingsdocumenten, wiki als gedeelde kennisbasis.

---

## Testomgeving

| Aspect | Waarde |
|--------|--------|
| Snapshot bij validatie | `Fase2-ZTNA-Complete` (basis), daarna Fase 3 incrementeel |
| pop01 RAM | 8 GB (verhoogd van 4 GB voor gelijktijdige Suricata + ClamAV-belasting) |
| ClamAV-signatures | daily v27953, main v63, bytecode v339, 3.642.437 signatures |
| Suricata-regels | ET Open + Abuse.ch: **79.620+** regels (Hyperscan-engine) |
| RPZ-records | URLhaus + ThreatFox: **71.767** records |
| mobile01 | Windows 11, VMware, extern; simuleert remote managed Windows client buiten schoolnetwerk |
| NetBird-overlaybereik | 100.64.0.0/10 |

---

## Gerelateerd

- [Architectuur](../overview/architecture.md)
- [Beslissing: SD-WAN Geschrapt](../decisions/sdwan-descoped.md)
- [Beslissing: CA + Posture hybride (Driepoortsmodel)](../decisions/ca-posture-hybrid.md)
- [Bevinding: curl --ssl-no-revoke op Windows](../findings/curl-ssl-no-revoke.md)
- [Bevinding: Suricata connection pooling](../findings/suricata-connection-pooling.md)
- [NetBird](../components/netbird.md)
- [Squid](../components/squid.md)
- [ClamAV/c-icap](../components/clamav-cicap.md)
- [Python DLP](../components/python-dlp.md)
- [Suricata](../components/suricata.md)
- [ioc2rpz](../components/ioc2rpz.md)

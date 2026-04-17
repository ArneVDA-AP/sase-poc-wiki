---
title: "Acceptance Test Results (F1–F15)"
tags: [sase, ztna, swg, fwaas, casb, sd-wan, testing]
---

# Acceptance Test Results (F1–F15)

**Version:** 1.0 — April 2026  
**Scope:** Full sandbox test coverage — all F1–F15 acceptance tests (Handboek v4 §46) plus additional validation tests outside the original framework.  
**Source:** `raw/SASE_PoC_Testrapport.md`

---

## Overall status

| Test | Name | Pillar | Status |
|------|------|--------|--------|
| **F1** | ZTNA Tunnel Connectivity | ZTNA | ✅ Validated |
| **F2** | Entra ID SSO | ZTNA | ✅ Validated |
| **F3** | Posture Check | ZTNA | ⏳ Planned — architecture ready (Addendum E) |
| **F4** | Datacenter Access via ZTNA | ZTNA | ✅ Validated |
| **F5** | URL Filtering | SWG | ✅ Validated |
| **F6** | SSL Bump Inspection | SWG | ✅ Validated |
| **F7** | Malware Detection (ClamAV) | SWG | ✅ Validated |
| **F8** | Firewall Segmentation | FWaaS | ✅ Validated (indirect — ZTNA ACL enforcement) |
| **F9** | Suricata Alert Generation | FWaaS | ✅ Validated (extended beyond original definition) |
| **F10** | Central Log Aggregation (SIEM) | SIEM | ⏳ Planned — requires Wazuh deployment |
| **F11** | CASB Alert and Remediation | CASB | ⏳ Planned — requires Wazuh + Graph API |
| **F12** | IPsec Tunnel Connectivity | SD-WAN | ✖ N/A — architectural decision (see [SD-WAN Descoped](../decisions/sdwan-descoped.md)) |
| **F13** | QoS Traffic Classification | SD-WAN | ✖ N/A — architectural decision |
| **F14** | Datacenter Access via SD-WAN | SD-WAN | ✖ N/A — architectural decision |
| **F15** | Full-Stack SASE Validation | Integration | ✅ Partial — steps 1–6 validated, steps 7–8 N/A (SD-WAN), step 9 planned (SIEM) |

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

---

## Coverage per SASE pillar

| Pillar | Validated | Planned | N/A |
|--------|-----------|---------|-----|
| **ZTNA** | F1, F2, F4 | F3 (architecture ready) | — |
| **SWG** | F5, F6, F7, T-A1, T-A2, T-A3, T-A4–A6 | — | — |
| **FWaaS** | F8, F9, T-A7, T-A8, T-A9 | F10 (Wazuh dep.) | — |
| **CASB** | — | F11 (Wazuh + Graph API) | — |
| **SD-WAN** | — | — | F12, F13, F14 |

---

## Validated tests — rationale and key output

### F1 — ZTNA Tunnel Connectivity

`InterfaceAlias: wt0` in the Test-NetConnection output proves that all traffic exits via the WireGuard tunnel, not the VMware host NIC. If wt0 were not routing, the reply would arrive via the default adapter.

```powershell
# mobile01 (PowerShell)
netbird status
Test-NetConnection 8.8.8.8 -Port 443
```

Expected output:

```
FQDN: mobile01.netbird.selfhosted
NetBird IP: 100.70.95.98/16
Peers count: 1/1 Connected

InterfaceAlias   : wt0          ← WireGuard tunnel, not host NIC
SourceAddress    : 100.70.95.98
TcpTestSucceeded : True
```

See [NetBird](../components/netbird.md).

---

### F2 — Entra ID SSO

Validates the full OIDC chain: mobile01 → NetBird → Zitadel → Entra ID (login.microsoftonline.com) → back to NetBird. The peer appearing in the NetBird Dashboard with the Entra ID username proves end-to-end identity linkage — not a local account.

Test procedure:

1. `netbird down` on mobile01
2. `netbird up` → browser redirects to `https://netbird.sandbox.local` → Zitadel → `login.microsoftonline.com`
3. Authenticate as `2ITCSC1A-mobile_user1@aplab.be`
4. Tunnel becomes active; peer appears in Dashboard, auto-assigned to `SASE-MobileUsers` group via setup key auto-group

See [NetBird](../components/netbird.md), [Decision: Zitadel as IdP broker](../decisions/zitadel-idp-broker.md).

---

### F3 — Posture Check (Planned)

Architecture fully specified in Addendum E (April 2026):

- **Gate 1:** Four Entra ID CA policies — MFA enforcement, geo-block (Belgium only), legacy auth block, sign-in risk
- **Gate 2:** Four NetBird posture checks — OS kernel ≥ 10.0.19041, client version, `MsMpEng.exe` AV process, geo Belgium
- Five planned validation scenarios, including negative tests (geo-block, OS version spoof, AV stopped, legacy auth, full positive end-to-end)

**Critical precondition:** Verify MFA registration on the test account before activating CA policies. Without prior MFA setup, the first login triggers an unrecoverable "MFA required but not configured" block, making the account inaccessible.

Scheduled for after the interim evaluation (20 April 2026). See [Decision: CA + Posture hybrid](../decisions/ca-posture-hybrid.md).

---

### F4 — Datacenter Access via ZTNA

DC-LAN (10.0.0.0/24) is only reachable through the NetBird Networks mechanism. A host not enrolled in NetBird has no route to this subnet regardless of IP connectivity.

```powershell
# mobile01 (PowerShell)
Test-NetConnection 10.0.0.100 -Port 80
```

Expected: `InterfaceAlias: wt0`, `TcpTestSucceeded: True`. Validates the `Datacenter Access` ACL policy and the Networks routing configuration.

See [NetBird](../components/netbird.md).

---

### F5 — URL Filtering

`curl.exe` with an explicit proxy argument bypasses the browser cache, which could otherwise mask blocking. `X-Squid-Error: ERR_ACCESS_DENIED` identifies the Squid block page — not a server-side 403.

```powershell
# mobile01 (PowerShell)
curl.exe -x http://100.70.154.79:3128 http://gambling.com -v
```

Expected:

```
< HTTP/1.1 403 Forbidden
< X-Squid-Error: ERR_ACCESS_DENIED 0

# Squid access log (pop01)
TCP_DENIED/403 ... GET http://gambling.com/ - HIER_NONE/-
```

Blocked categories: adult, malware, phishing, gambling (UT1 Toulouse Remote ACL) plus manual blacklist entries (`gambling.com`, `.bet365.com`, `.pokerstars.com`). See [Squid](../components/squid.md).

---

### F6 — SSL Bump Inspection

Two complementary tests prove selective, intentional HTTPS interception:

- **Normal HTTPS:** Navigate to `https://google.com` → certificate issuer = `O = SASE PoC` (not Google)
- **No-bump exception:** Navigate to `https://login.microsoftonline.com` → issuer = Microsoft Corporation (original certificate, not replaced)

The second test is critical: it proves the implementation is deliberate, not blanket decryption. Microsoft login is in the no-bump list specifically because SSL Bump would break the Entra ID OIDC flow.

See [SSL Bump](../concepts/ssl-bump.md), [Squid](../components/squid.md).

---

### F7 — Malware Detection (ClamAV/EICAR)

The 68-byte vs. 8245-byte comparison is the proof of active blocking: 68 bytes is the EICAR file; 8245 bytes is the Squid block page generated by the ICAP pipeline.

```powershell
# mobile01 (PowerShell)
curl.exe -x http://100.70.154.79:3128 --ssl-no-revoke -o eicar_test.txt https://secure.eicar.org/eicar.com
```

> `--ssl-no-revoke` is required on Windows: Schannel attempts CRL/OCSP validation against the SASE-PoC-CA certificate, which has no published CRL endpoint. See [Finding: curl --ssl-no-revoke](../findings/curl-ssl-no-revoke.md).

Expected:

```
# c-icap log (pop01)
VIRUS DETECTED: Eicar-Test-Signature,
  http client ip: 100.70.95.98,
  http url: https://secure.eicar.org/eicar.com

# Squid access log
TCP_MISS/403 8245 GET https://secure.eicar.org/eicar.com
```

See [ClamAV/c-icap](../components/clamav-cicap.md).

---

### F8 — Firewall Segmentation

DC-LAN isolation is enforced at the overlay layer — only NetBird-enrolled peers with the `Datacenter Access` ACL policy can reach dc01.

Validated by contrast:

- mobile01 **without** NetBird: `ping 10.0.0.100` → Destination unreachable (no route)
- mobile01 **with** NetBird + policy: `Test-NetConnection 10.0.0.100 -Port 80` → `TcpTestSucceeded: True` via `wt0`

**Architectural note:** DC-LAN uses NetBird Networks (not Network Routes). Access requires both overlay enrollment and explicit group membership in `SASE-InternalResources`.

See [NetBird](../components/netbird.md).

---

### F9 — Suricata Alert Generation

Four test categories validate four independent inspection domains across two interfaces:

| Test | Method | SID | Interface |
|------|--------|-----|-----------|
| F9-1: HTTP content | `curl.exe -x ... http://testmyids.com/` | 2100498 (GPL ATTACK_RESPONSE id check returned root) | vtnet0 |
| F9-2: DNS anomaly | Automatic — Unbound `.biz` TLD queries during normal operation | 2027863 (ET DNS Non-Compliant DNS Reply UDP) | vtnet0 |
| F9-3: User-Agent | `curl.exe -x ... -A "BlackSun" http://example.com` | 2008983 (ET MALWARE User-Agent BlackSun) | vtnet0 |
| F9-4: LAN traffic | `apt update` on dc01 | 2013504 (ET INFO GNU/Linux APT User-Agent Outbound) | vtnet1 |

vtnet1 verification:

```bash
grep '"in_iface":"vtnet1".*"event_type":"alert"' /var/log/suricata/eve.json | wc -l
# Expected: > 0
```

**Differentiated alert policy:**

| Category | Policy | Rationale |
|----------|--------|-----------|
| emerging-malware, botcc, C2, Abuse.ch | Drop | Always malicious — no legitimate traffic possible |
| tor, info, policy, dns, web | Alert | False positive risk — monitor, do not block |

> The Handboek originally defined F9 as "verify alert visible in Wazuh Dashboard." Wazuh is not yet deployed (Fase 4 — F10). Suricata detection is fully validated; the Wazuh integration is the only gap.

> Multiple curl requests to the same destination generate one Suricata alert per SID per flow — this is correct behavior, not suppression. Root cause: Squid connection pooling reuses upstream TCP connections; Suricata sees one flow. See [Finding: Suricata connection pooling](../findings/suricata-connection-pooling.md).

See [Suricata](../components/suricata.md).

---

### T-A1 — DLP YARA: CONFIDENTIAL label in download

YARA operates after ClamAV's file decomposition engine — it matches on extracted document text, not raw bytes. Full pipeline: Squid SSL Bump → c-icap RESPMOD → ClamAV parse → YARA match → HTTP 403.

```bash
# pop01 — local verification
clamdscan /tmp/dlp_test.txt   # file contains "Dit document is CONFIDENTIAL..."
# Output: YARA.DLP_Confidential_Label.UNOFFICIAL FOUND
```

```powershell
# mobile01 — end-to-end via ICAP
curl.exe -x http://100.70.154.79:3128 --ssl-no-revoke -o dlp_result.txt http://wpad.sandbox.local/dlp_test.txt -w "%{http_code}"
# Output: 403
```

YARA rules validated in sandbox:

| Rule | Pattern | Scope |
|------|---------|-------|
| `DLP_Confidential_Label` | CONFIDENTIAL, VERTROUWELIJK, GEHEIM, DO NOT DISTRIBUTE (nocase) | ✅ End-to-end |
| `DLP_IBAN_Pattern` | NL/DE/BE IBAN format regex | ✅ Local clamdscan |
| `DLP_BSN_Candidate` | Cluster of 9-digit sequences (threshold >2) | ✅ Local clamdscan |
| `DLP_AWS_AccessKey` | `AKIA[0-9A-Z]{16}` | ✅ Local clamdscan |

**Limitation:** YARA rules do not perform algorithmic validation — no mod-97 for IBAN, no 11-proof for BSN. False positives are possible on download. The Python DLP layer (T-A3) provides algorithmic validation for uploads.

See [ClamAV/c-icap](../components/clamav-cicap.md), [DLP](../concepts/dlp.md).

---

### T-A2 — DLP SDD: StructuredDataDetection threshold

ClamAV's StructuredDataDetection (SDD) uses Luhn validation — arbitrary 16-digit strings do not trigger. The threshold is `StructuredMinCreditCardCount: 3` (configured), so detection fires at 4+ valid CC numbers.

```bash
# 4 valid Luhn CC numbers → detected
echo "CC1: 4532015112830366 CC2: 4916338506082832 CC3: 5425233430109903 CC4: 2223000048410010" > /tmp/sdd_test.txt
clamdscan /tmp/sdd_test.txt
# Output: Heuristics.Structured.CreditCardNumber FOUND

# 1 valid CC number → passes (below threshold)
echo "CC: 4532015112830366" > /tmp/sdd_test_single.txt
clamdscan /tmp/sdd_test_single.txt
# Output: OK
```

See [ClamAV/c-icap](../components/clamav-cicap.md).

---

### T-A3 — Python DLP ICAP REQMOD: upload CC blocking

ClamAV c-icap cannot parse `multipart/form-data` POST bodies — this is a confirmed upstream limitation (SourceForge: not implemented in the `virus_scan` service). Python DLP fills this gap via ICAP REQMOD.

```powershell
# mobile01 (PowerShell)
curl.exe -x http://100.70.154.79:3128 --ssl-no-revoke -X POST `
  -d "payment_info=4532015112830366&name=Test+User" `
  https://httpbin.org/post
# Expected: HTML DLP block page, HTTP 403
```

The DLP block page is distinguishable from Squid's generic 403 by its DLP-specific error message.

ICAP OPTIONS verification (confirms REQMOD is active):

```bash
# pop01 (FreeBSD) — use printf, not echo -e (FreeBSD /bin/sh does not interpret -e as escape flag)
printf "OPTIONS icap://192.168.122.23:1345/dlpscan ICAP/1.0\r\nHost: 192.168.122.23\r\n\r\n" | nc -w 5 192.168.122.23 1345
# Output: ICAP/1.0 200 OK
#         Methods: REQMOD
```

Validation matrix:

| Input | Algorithm | Status |
|-------|-----------|--------|
| `4532015112830366` (CC) | Luhn | ✅ End-to-end validated |
| `NL91ABNA0417164300` (IBAN) | mod-97 | ✅ Code present, not end-to-end tested |
| `111222333` (BSN) | 11-proof | ✅ Code present, not end-to-end tested |

See [Python DLP](../components/python-dlp.md), [Decision: Two-Layer DLP](../decisions/two-layer-dlp.md).

---

### T-A4, T-A5, T-A6 — DNS RPZ: three network segments

Test domain: `testentry.rpz.urlhaus.abuse.ch` — a permanent abuse.ch validation entry, always present in the URLhaus RPZ feed.

The `aa` (authoritative answer) flag is the key discriminator: a real internet NXDOMAIN lacks `aa`; an RPZ block is answered authoritatively by the local resolver without an upstream query.

Control test confirms no over-blocking: `drill @127.0.0.1 google.com` returns `NOERROR` without the `aa` flag.

| Segment | Command | Result |
|---------|---------|--------|
| **T-A4** pop01 (local) | `drill @127.0.0.1 testentry.rpz.urlhaus.abuse.ch` | `rcode: NXDOMAIN, flags: aa` |
| **T-A5** mobile01 (overlay) | `nslookup testentry.rpz.urlhaus.abuse.ch` | Non-existent domain |
| **T-A6** dc01 (DC-LAN) | `drill @10.0.0.1 testentry.rpz.urlhaus.abuse.ch` | `rcode: NXDOMAIN, flags: aa` |

Unbound RPZ log confirms the zone name and source IP for each query:

```
rpz: applied [ioc2rpz-threat-intel] testentry.rpz.urlhaus.abuse.ch.
  rpz-nxdomain 100.70.95.98@58107 testentry.rpz.urlhaus.abuse.ch. A IN
```

DNS path for T-A5: mobile01 → NetBird DNS relay (100.70.255.254) → pop01 Unbound → RPZ check → NXDOMAIN.

See [ioc2rpz](../components/ioc2rpz.md), [RPZ](../concepts/rpz.md).

---

## Planned tests and blockers

### F10 — Central Log Aggregation (SIEM)

Requires Wazuh deployment (Fase 4). Log sources already available: Squid access log, Suricata `eve.json`, OPNsense firewall log, Python DLP Docker log. Wazuh-Suricata integration via the Wazuh OPNsense module is standard and requires no custom development.

### F11 — CASB Alert and Remediation

Requires Wazuh deployment and Microsoft Graph API configuration. Architecture is designed (Afbakening v1.2 §2.2):

- **Inline layer:** Squid + ICAP (already operational)
- **API layer:** Wazuh `ms-graph` module + Graph API (`AuditLog.Read.All`, `Directory.Read.All`)
- **Custom rules:** SID 100200/100201 for SharePoint `SharingSet` + `Anyone`/`External` detection
- **Active Response:** `sharepoint_remediate.sh`

Graph API tenant permissions confirmed 1 April 2026 (admin consent granted on `aplab.be`). Only remaining blocker: Wazuh deployment.

---

## F15 — Full-Stack Validation step-by-step

| Step | Description | Status |
|------|-------------|--------|
| 1 | Mobile user logs in via Entra ID | ✅ Validated (F2) |
| 2 | Visits HTTPS site → SSL Bump active | ✅ Validated (F6) |
| 3 | Visits blocked site → Blocked | ✅ Validated (F5) |
| 4 | Downloads EICAR → Blocked | ✅ Validated (F7) |
| 5 | Visits testmyids.com → Alert in IDS | ✅ Validated (F9-1) |
| 6 | Opens datacenter site → Success | ✅ Validated (F4) |
| 7 | Site user pings datacenter via tunnel | ✖ N/A — sitepc01 has no OS, IPsec descoped |
| 8 | QoS marking visible on VyOS | ✖ N/A — QoS architectural decision |
| 9 | Management dashboard shows all services UP | ⏳ Planned (Wazuh) |

6 of 9 F15 steps validated. The 3 remaining are either architectural N/A (SD-WAN) or planned (SIEM).

---

## Test environment

| Aspect | Value |
|--------|-------|
| Snapshot at validation | `Fase2-ZTNA-Complete` (base), then Phase 3 incremental |
| pop01 RAM | 8 GB (increased from 4 GB for concurrent Suricata + ClamAV load) |
| ClamAV signatures | daily v27953, main v63, bytecode v339 — 3,642,437 signatures |
| Suricata rules | ET Open + Abuse.ch: **79,620+** rules (Hyperscan engine) |
| RPZ records | URLhaus + ThreatFox: **71,767** records |
| mobile01 | Windows 11, VMware, external — simulates BYOD outside school network |
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

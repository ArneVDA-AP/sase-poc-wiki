---
title: "Attack & Bypass Scenarios (Demo Validation)"
tags: [testing, sase, demo, swg, ztna, casb, fwaas, ids, dlp, rpz]
---

# Attack & Bypass Scenarios (Demo Validation)

These scenarios validate each SASE pillar by attempting attacks or policy bypasses that the stack should detect and block. Each scenario maps to one or more acceptance tests (F1--F15, T-A1--T-A13) documented in [Acceptance Tests](acceptance-tests.md).

---

## Scenarios by pillar

### ZTNA -- Identity & Tunnel

| # | Scenario | Expected result | Validates | Test command |
|---|----------|-----------------|-----------|--------------|
| A1 | Unenrolled device attempts DC-LAN access | No route to 10.0.0.0/24 -- destination unreachable | F8 | `ping 10.0.0.100` from unenrolled host |
| A2 | NetBird login via Entra ID | OIDC flow completes, tunnel active, peer in correct group | F1, F2 | `netbird up` followed by browser SSO |
| A3 | Non-compliant device login | Conditional Access blocks (report-only until demo) | F3 | Modify device compliance in Intune |

### SWG -- Web Gateway Inspection

| # | Scenario | Expected result | Validates | Test command |
|---|----------|-----------------|-----------|--------------|
| B1 | Browse to blocked URL category | Squid 403 ERR_ACCESS_DENIED | F5 | `curl.exe -x http://100.70.154.79:3128 http://gambling.com` |
| B2 | EICAR malware download | ClamAV blocks, Squid 403 (8245 bytes vs 68) | F7 | `curl.exe -x ... --ssl-no-revoke https://secure.eicar.org/eicar.com` |
| B3 | SSL Bump verification | Certificate issuer = SASE-PoC-CA, not original | F6 | Browser navigate to google.com, inspect certificate |
| B4 | No-bump exception (Microsoft login) | Certificate issuer = Microsoft (original) | F6 | Browser navigate to login.microsoftonline.com, inspect certificate |
| B5 | DLP upload -- credit card in POST body | Python DLP ICAP blocks, 403 | T-A3 | `curl.exe -x ... -X POST -d "CC: 4532015112830366"` to httpbin |
| B6 | DLP download -- CONFIDENTIAL label | ClamAV YARA blocks | T-A1 | Download file containing CONFIDENTIAL marker |
| B7 | DLP threshold -- 4x CC numbers block, 1x passes | SDD threshold enforcement | T-A2 | POST with 4 vs 1 credit card numbers |

### CASB -- Identity-Based Filtering

| # | Scenario | Expected result | Validates | Test command |
|---|----------|-----------------|-----------|--------------|
| C1 | Student browses ChatGPT | Blocked (Studenten policy) | CASB L1 | `curl.exe -x ... https://chatgpt.com` as student |
| C2 | Teacher browses ChatGPT | Allowed (Docenten policy) | CASB L1 | Same URL as teacher identity |
| C3 | SharePoint anonymous share | Wazuh Active Response revokes link | CASB L2 | Create anonymous sharing link in SharePoint |

### FWaaS / IDS -- Network Detection

| # | Scenario | Expected result | Validates | Test command |
|---|----------|-----------------|-----------|--------------|
| D1 | testmyids.com attack response | Suricata SID 2100498 (GPL ATTACK_RESPONSE) | F9 | `curl.exe -x ... http://testmyids.com/` |
| D2 | Suspicious User-Agent "BlackSun" | Suricata SID 2008983 (ET MALWARE) | T-A8 | `curl.exe -x ... -A "BlackSun" http://example.com` |
| D3 | DNS anomaly detection | Suricata SID 2027863 | T-A9 | Automatic via .biz TLD queries |
| D4 | DC-LAN traffic detection (vtnet1) | Suricata alert on vtnet1 | T-A7 | `apt update` on dc01 |

### DNS Threat Intelligence

| # | Scenario | Expected result | Validates | Test command |
|---|----------|-----------------|-----------|--------------|
| E1 | Query RPZ-blocked domain from pop01 | NXDOMAIN with aa flag | T-A4 | `dig @127.0.0.1 <rpz-domain>` on pop01 |
| E2 | Query RPZ-blocked domain from mobile01 | NXDOMAIN via overlay DNS | T-A5 | `nslookup <rpz-domain>` on mobile01 |
| E3 | Query RPZ-blocked domain from dc01 | NXDOMAIN via DC-LAN | T-A6 | `dig @10.0.0.1 <rpz-domain>` on dc01 |

### Event-Driven Enforcement (NATS)

| # | Scenario | Expected result | Validates | Test command |
|---|----------|-----------------|-----------|--------------|
| F1 | High-severity IDS alert | Control daemon quarantines peer (<500 ms) | CASB L3 | Trigger Suricata C2 alert |
| F2 | Multiple medium alerts (score accumulation) | Threat score rises, crosses threshold, triggers quarantine | CASB L3 | Sequential alerts from same client |
| F3 | Score decay after quarantine | Peer restored to persona group after decay period | CASB L3 | Wait for sliding-window decay |

---

## Important test notes

- All proxy tests from Windows require `--ssl-no-revoke` due to Schannel CRL check failure on SASE-PoC-CA (see [Finding: curl --ssl-no-revoke](../findings/curl-ssl-no-revoke.md)).
- ICMP ping to overlay peers fails by design under the ALLOW-only policy model -- always use port-based tests.
- Each Suricata SID fires once per TCP flow due to connection pooling (see [Finding: Suricata connection pooling](../findings/suricata-connection-pooling.md)).

---

## Related

- [Acceptance Tests (F1--F15)](acceptance-tests.md)
- [Architecture](../overview/architecture.md)
- [Concept: Zero Trust](../concepts/zero-trust.md)
- [Finding: curl --ssl-no-revoke](../findings/curl-ssl-no-revoke.md)
- [Finding: Suricata connection pooling](../findings/suricata-connection-pooling.md)

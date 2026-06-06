---
title: "Testing: Transparent Proxy and Enforcement"
tags: [testing, tproxy, proxy, enforcement, intune, sase]
---

# Testing: Transparent Proxy and Enforcement

**Scope:** Diagnostic validation of the transparent-proxy research (Option C rejected, Option D grounded architecturally) plus the managed-device enforcement variant that runs live on the sandbox.  
**Source:** TransProxy_Verslag02, Addendum F v3, Verslag41.

---

## Position within the acceptance tests

This test block falls outside the original F1-F15 series but supplies evidence for the same rubric criteria. The tests fall into two categories:

1. **T-TP series** (Transparent Proxy): diagnostic tests that underpin the architecture choice. Run on the `.11/.17` environment (the parallel stack), not repeated on the sandbox (Session 9 deprioritised).
2. **T-ME series** (Managed Enforcement): the endpoint-enforcement variant that is live on the sandbox (V41). This is the production path for managed devices.

The full component lives at [Transparent Proxy: TPROXY on linuxpop01](../components/transparent-proxy.md).

---

## Overall status

| Test | Name | Status |
|---|---|---|
| **T-TP1** | Source IP preserved after masquerade OFF | ✅ Validated (`.11` environment) |
| **T-TP2** | pf rdr matches overlay traffic on vtnet0 | ✅ Validated (`.11` environment) |
| **T-TP3** | `SYN_RCVD` hang: return path asymmetric under source-IP preservation | ✅ Validated (`.11` environment); proves Option C fails |
| **T-TP4** | Masquerade ON masks the problem | ✅ Validated (`.11` environment); explains why earlier tests passed |
| **T-TP5** | TPROXY source-IP preservation (Option D, test 5) | ✖ Not executed; Session 9 deprioritised; grounded architecturally (`IP_TRANSPARENT` kernel property) |
| **T-TP6** | Symmetric return path (Option D, test 10) | ✖ Not executed; Session 9 deprioritised; structurally guaranteed (interception and termination on the same node) |
| **T-ME1** | Intune proxy CSP: bump + overlay `%SRC` in access.log | ✅ Validated (sandbox, V41) |
| **T-ME2** | Intune firewall CSP: QUIC→TCP fallback + SSL Bump | ✅ Validated (sandbox, V41) |
| **T-ME3** | Intune firewall CSP: tunnel + web + MDM intact | ✅ Validated (sandbox, V41) |
| **T-ME4** | DLP block end-to-end via the enforced proxy | ✅ Validated (sandbox, V41) |

> Option D (T-TP5/T-TP6) is **designed, not tested**. The diagnostics (T-TP1-T-TP4) are empirically validated on the parallel stack; the two TPROXY tests were not run because linuxpop01 does not exist on the sandbox and Session 9 was deprioritised. The expected result follows from a documented kernel property, not from a measurement.

---

## Coverage per rubric criterion

### SWG: Traffic routing (no local breakout)

**Rubric criterion:** fully forced through the SWG.

| Test | What it proves |
|---|---|
| T-TP3 | Diagnostic evidence of *why* a centralised pf rdr fails: the architectural grounding for choosing TPROXY or endpoint enforcement. |
| T-ME1 | The Intune-pushed proxy is system-wide for all WinINET-respecting apps; not something the user can undo (non-admin). |
| T-ME2 | QUIC traffic is blocked by the firewall CSP; the browser falls back to TCP/443 → Squid bumps it anyway. |
| T-ME3 | Firewall rules break no legitimate traffic: the NetBird tunnel, web browsing via the proxy, and MDM check-in stay intact. |

### SWG: Malware inspection (DPI)

**Rubric criterion:** DPI + TLS decryption + malware detection.

| Test | What it proves |
|---|---|
| T-ME4 | DLP ICAP pipeline live: HTTPS destination bumped (`O=SASE PoC`), DPI on the payload, blocked with a readable reason (`YARA.DLP_Confidential_Label`). Proves end-to-end: TLS decryption → DPI → block. |
| T-ME2 | The QUIC block forces a TCP/443 fallback → Squid bumps → ClamAV/DLP inspect the full payload. |

### General: Troubleshooting

**Rubric criterion:** in-depth analysis.

| Test | What it proves |
|---|---|
| T-TP1-T-TP4 | Layered diagnostics: each layer of the stack validated individually (`tcpdump`, `pfctl -vvs nat`, `netstat -an`, conntrack analysis). Root cause isolated via two simultaneous tcpdumps (ens3 versus wt0). A wrong diagnosis (01.9) corrected on the basis of empirical evidence. |

### General: Architecture

**Rubric criterion:** professional.

| Evidence | What it proves |
|---|---|
| Five-option analysis | Systematic evaluation of architecture options with pros/cons per option. |
| Option C implementation + rejection | Hypothesis tested, refuted, and corrected: an architecture decision on the basis of evidence, not assumption. |
| Managed-device pivot | Architectural restructuring on the basis of changed assumptions (Intune enrollment makes network interception not required for managed devices). |

### General: Collaboration

**Rubric criterion:** proactive collaboration, knowledge sharing, and joint problem solving.

| Evidence | What it proves |
|---|---|
| The teammate's finding + decision doc | Problem identification and first architecture proposal by a team member. |
| TransProxy_Verslag02 | The diagnostics build on the teammate's implementation; correct his wrong diagnosis (01.9) but validate his architecture choice (Option B). |
| Revert of the `.11` environment | After the diagnostics, restored to the working state (masquerade ON): no regression for the team. |

---

## Test details

### T-TP1: source IP preserved after masquerade OFF

**Goal:** prove the overlay IP stays intact after removing the NetBird masquerade.  
**Precondition:** NetBird exit node recreated as a Network Route (`0.0.0.0/0`, masquerade OFF).  
**Command:** `tcpdump -i ens3 -n 'tcp port 443'` on linuxpop01 during `curl` from mobile01.  
**Acceptance criterion:** `src=100.86.x.x` (overlay IP), not `192.168.122.17` (linuxpop01 LAN IP).  
**Measured result:** ✅ Source IP preserved. Confirms that finding 01.9 ("Linux rewrites the source IP") was wrong: the masquerade sat in nftables `netbird-rt-postrouting`.

### T-TP2: pf rdr matches overlay traffic on vtnet0

**Goal:** confirm the OPNsense pf rdr intercepts overlay traffic correctly after it arrives on vtnet0.  
**Command:** `pfctl -vvs nat` on pop01.  
**Acceptance criterion:** state creations > 0 on the rdr rules for `100.64.0.0/10`.  
**Measured result:** ✅ 536 state creations on the HTTPS rule. pf rdr works correctly: the blocker is not here.

### T-TP3: `SYN_RCVD` hang, return path asymmetric

**Goal:** prove the return path is structurally broken under source-IP preservation with interception behind a second stateful router.  
**Commands (simultaneous):**

- `netstat -an | grep SYN_RCVD` on pop01
- `tcpdump -i ens3 -n 'src 192.168.122.11'` on linuxpop01
- `tcpdump -i wt0 -n` on linuxpop01

**Acceptance criterion:** the SYN-ACK arrives on ens3 but does not appear on wt0 (conntrack mismatch → DROP).  
**Measured result:** ✅ Exactly as predicted. This is the decisive evidence that Option C cannot work architecturally under source-IP preservation. It is not a configuration error, but a fundamental property of the netfilter conntrack state machine.

### T-TP4: masquerade ON masks the problem

**Goal:** explain why the implementation with masquerade ON did work.  
**Analysis:** with masquerade ON the source was `192.168.122.17`, linuxpop01's local IP. Squid's reply went to a local IP → local delivery → linuxpop01's conntrack de-NAT'd it back to the client. No forwarding path, so no conntrack mismatch.  
**Measured result:** ✅ Explains the mechanism. The price: source IP lost → Identity Bridge unusable.

### T-TP5: TPROXY source-IP preservation (Option D)

**Goal:** prove that the Squid on linuxpop01 sees the real client overlay IP via TPROXY.  
**Command:** `tail -f /var/log/squid/access.log` on linuxpop01 during browsing from mobile01.  
**Derived result:** `src = 100.86.x.x` (not the linuxpop01 IP).  
**Status:** ✖ Not executed. linuxpop01 does not exist on the sandbox; Session 9 was deprioritised because managed-device enforcement (V41) covers the primary path. The proposed solution: Squid with TPROXY on linuxpop01 intercepts in `iptables mangle PREROUTING` on the `wt0` interface, before the forwarding decision. The `IP_TRANSPARENT` socket option preserves both src and dst by definition; this is a documented kernel property, not a hypothesis. See [Transparent Proxy: TPROXY on linuxpop01](../components/transparent-proxy.md) for the full architecture and implementation specification.

### T-TP6: symmetric return path (Option D)

**Goal:** prove the return path is now symmetric with TPROXY (the problem that sank Option C).  
**Command:** `tcpdump -i wt0 -n` on linuxpop01 during `curl` from mobile01.  
**Derived result:** the SYN-ACK appears back on wt0 (no `SYN_RCVD` hang).  
**Status:** ✖ Not executed (same reason as T-TP5). This result follows logically from the architecture: the Squid on linuxpop01 is both interception point and connection endpoint; the packet is steered to the local socket in `mangle PREROUTING` and never leaves the forwarding pipeline. The asymmetric return-path mismatch that T-TP3 proved (conntrack on an intermediate router drops the SYN-ACK) cannot occur here structurally. See [The diagnostics: why Option C fails](../components/transparent-proxy.md#the-diagnostics-why-option-c-fails) for the contrast.

### T-ME1: Intune proxy CSP, bump + overlay `%SRC` in access.log

**Goal:** prove the Intune-pushed proxy works system-wide and passes the overlay IP through.  
**Commands:**

- On mobile01: browse to `google.com`, check the certificate issuer.
- On pop01: `tail -f /var/log/squid/access.log`.

**Acceptance criterion:** issuer = `O=SASE PoC` (bump active); access.log shows `100.70.178.122` (overlay IP, not a LAN IP).  
**Measured result:** ✅ Validated (V41 41.11). Microsoft domains spliced correctly (no-bump).

### T-ME2: Intune firewall CSP, QUIC→TCP fallback + SSL Bump

**Goal:** prove the Intune firewall block of QUIC (UDP/443) forces browsers onto TCP/443, where Squid inspects it.  
**Command:** browse to `google.com` (Google is a QUIC property); check the certificate issuer.  
**Acceptance criterion:** certificate issuer = `O=SASE PoC`; the browser fell back to TCP/443 and Squid bumped it.  
**Measured result:** ✅ Validated (V41 41.19/41.20). The bump proves QUIC was blocked.

### T-ME3: Intune firewall CSP, tunnel + web + MDM intact

**Goal:** prove the firewall blocks (QUIC/DoT) break no legitimate traffic.  
**Checks:**

1. `netbird status` → Connected (the tunnel survives; the UDP/443 block does not touch the 51820 tunnel).
2. `google.com` loads in the browser (web via the proxy intact).
3. Intune sync successful (MDM check-in via the WinHTTP SYSTEM path open).

**Acceptance criterion:** all three green.  
**Measured result:** ✅ Validated (V41 41.19). Three B3 criteria proven: "the firewall breaks no legitimate traffic".

### T-ME4: DLP block end-to-end via the enforced proxy

**Goal:** prove the full DPI chain works via the Intune-enforced proxy.  
**Measured result (incidentally during T-ME2):** the DLP ICAP pipeline logged a blocked transfer live:

```
malware: YARA.DLP_Confidential_Label.UNOFFICIAL
HTTP location: https://docs.cloud.google.com/...
```

This proves end-to-end: HTTPS destination bumped (otherwise ICAP could not see the payload) → DPI + TLS decryption → blocked with a readable reason.  
**Assessment:** ✅ Validated (V41 41.20). Touches rubric SWG malware inspection ("DPI + TLS decryption + malware detection") and General transparency ("explanation of why it was blocked").

---

## Related

- [Transparent Proxy: TPROXY on linuxpop01](../components/transparent-proxy.md)
- [Acceptance Tests (F1-F15)](acceptance-tests.md)
- [Attack & Bypass Scenarios](attack-scenarios.md)

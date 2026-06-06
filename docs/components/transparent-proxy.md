---
title: "Transparent Proxy: TPROXY on linuxpop01"
tags: [tproxy, squid, proxy, netbird, enforcement, sase, identity-bridge, intune]
---

# Transparent Proxy: TPROXY on linuxpop01

**Role:** Defense-in-depth enforcement for unmanaged (BYOD) devices — transparent interception of all HTTP/HTTPS traffic arriving over the NetBird overlay, regardless of client configuration.  
**Status:** ✅ PoC-validated on the parallel stack (diagnostics); Option D designed, sandbox integration pending. The architecture choice is validated on the `.11` environment but not yet integrated into the sandbox whole: Session 9 was deprioritised because managed-device enforcement via Intune covers the primary path (V41).  
**Node:** linuxpop01 (Ubuntu 24.04). On the parallel stack this is `192.168.122.17`; the proposed sandbox node takes `192.168.122.43` and has not been created yet.

---

## Problem statement

The sandbox uses an [explicit proxy via WPAD/PAC](../decisions/wpad-vs-transparent-proxy.md): clients send their HTTP/HTTPS traffic to Squid on `100.70.154.79:3128` because the PAC file tells them to. This is a client-side compliance model. Inspection only works if the application respects the system proxy setting.

The problem is applications with their own network stack (Spotify, Teams system services, game launchers, PowerShell scripts). They ignore the WinINET proxy and send their traffic straight through the NetBird tunnel. That traffic exits uninspected through the masqueraded tunnel: no Squid log, no DLP, no URL filter, no Identity Bridge trace.

This hits the rubric on two points:

| Rubric criterion | Impact |
|---|---|
| **SWG, Traffic routing** | "Fully forced through the SWG" requires that *all* traffic passes the inspection stack, not just browser traffic. |
| **SWG, Identity-based access** | The Identity Bridge (`external_acl_type` with `%SRC`) only works for traffic that passes Squid. Non-proxy traffic is invisible. |

The fundamental question: how do you enforce inspection on traffic that ignores the proxy?

---

## Options considered

The research analysed five architecture options. Option B was operational on the `.11` environment (the working implementation with masquerade ON). Option C was thoroughly implemented and tested in several configurations on the same environment; the diagnostics that followed are the core of this page. The full analysis lives in Addendum F v2 (five-option analysis) and Addendum F v3 (revision after the diagnostics).

### Option A: TPROXY + cache_peer (double Squid)

linuxpop01 intercepts via TPROXY and forwards to the pop01 Squid via `cache_peer`.

**Rejected.** SSL Bump + cache_peer is fundamentally incompatible. Two scenarios, both broken: (1) both Squids bump, so double MITM and TLS breaks; (2) only pop01 bumps, so linuxpop01 cannot inspect the CONNECT tunnel. Not solvable architecturally without breaking the SSL Bump chain.

### Option B: TPROXY standalone on linuxpop01

linuxpop01 runs its own Squid with TPROXY (`IP_TRANSPARENT`), SSL Bump, and ICAP. The Squid on linuxpop01 is both interception point and connection endpoint.

**Architecturally correct**, the original choice. The price: a second Squid instance with its own SSL Bump and ICAP configuration. In v2 this option was passed over in favour of Option C to keep a single Squid. In v3 Option C turned out to be untenable and Option B (renamed Option D with refinements) became the chosen architecture after all.

### Option C: routing-only gateway + pf rdr on pop01

linuxpop01 as a lightweight NAT gateway (`ip_forward` only, no Squid), with transparent `pf rdr` interception on pop01 (OPNsense).

**Rejected after implementation and diagnostics.** Option C was thoroughly implemented and tested on the `.11/.17` environment. The diagnostics produced a complete causal model for the failure. See [The diagnostics: why Option C fails](#the-diagnostics-why-option-c-fails).

### ~~mgmt01 as exit node~~ (rejected)

**Rejected on an architecture principle.** It mixes the control plane (NetBird management, Zitadel, Caddy) with the data plane (exit node). A violation of the Zero Trust blast-radius principle.

### ~~NetBird route without exit node~~ (rejected)

A variant of Option C. Same routing path, same return-path problem.

---

## The diagnostics: why Option C fails

*Source: TransProxy_Verslag02 (30-31 May 2026), the full diagnostic session on the parallel stack.*

The diagnostics started as a review of the working implementation (masquerade ON) and escalated into a full unravelling of the return-path problem. The core finding: transparent proxy interception on a node behind a second stateful router cannot return traffic correctly once the source IP is preserved.

### Step 1: trace the source-IP loss

Finding 01.9 ("Linux rewrites the source IP on forwarding") turned out to be misdiagnosed. Linux forwarding never rewrites source IPs without explicit SNAT or MASQUERADE. The real cause: NetBird's "Add Exit Node" feature forces masquerade ON in an nftables chain (`netbird-rt-postrouting`), with no UI toggle. The NetBird documentation states this verbatim.

**Fix:** remove the exit node and recreate it as a regular Network Route (`0.0.0.0/0`, masquerade OFF).

### Step 2: masquerade OFF, pf rdr matches

After removing the masquerade, the overlay IP (`100.86.x.x`) arrives intact on pop01 vtnet0. The `pf rdr` rules match correctly:

```
@6 rdr on vtnet0 inet proto tcp from 100.64.0.0/10 ... port = https -> 192.168.122.11 port 3130
   [ Packets: 10275  State Creations: 536 ]
```

### Step 3: Squid receives the SYN, but the client gets nothing back

The TCP state on pop01 during a connection:

```
tcp4  192.168.122.11:3129  100.86.233.213:57989  SYN_RCVD
```

`SYN_RCVD` is the decisive evidence: Squid received the SYN, the kernel sent a SYN-ACK, but the client's ACK never arrives. Two simultaneous tcpdumps on linuxpop01 confirmed it: the SYN-ACK arrives untranslated on ens3 (`src=192.168.122.11:3129`) but never appears on wt0. Conntrack drops it.

### Step 4: the causal model

```
Squid replies from 192.168.122.11:3129
  → pf does NOT reverse-translate (locally terminated connection)
  → SYN-ACK leaves untranslated: 192.168.122.11:3129 → 100.86.233.213
  → Routes via static route to linuxpop01
  → On linuxpop01: conntrack expects a reply from origin:80, gets 192.168.122.11:3129
  → INVALID → FORWARD RELATED,ESTABLISHED → DROP
  → Client gets nothing → SYN_RCVD hang → timeout
```

This is an architectural mismatch, not a configuration error. The interception sits on the wrong node: a pf rdr behind a second stateful router (linuxpop01's conntrack) cannot return symmetrically. This is exactly why Linux TPROXY exists.

### Why masquerade ON masked it

With masquerade ON, the source was `192.168.122.17` (linuxpop01's own IP). Squid's reply went to a local IP, so local delivery and no forwarding. linuxpop01's masquerade conntrack de-NAT'd it back to the client. It worked not *through* the masquerade, but because the source happened to be local. The price: source IP lost, and with it the Identity Bridge unusable.

---

## Chosen architecture: Option D (TPROXY on linuxpop01)

*Source: Addendum F v3, TPROXY Revision (31 May 2026).*

```
mobile01
  └─[WireGuard]──► linuxpop01 (wt0)
                     │
                     │ iptables mangle PREROUTING
                     │   TPROXY: TCP/80  → local Squid :3129
                     │   TPROXY: TCP/443 → local Squid :3130
                     ▼
                   Squid on linuxpop01
                     │  sees ORIGINAL src (100.86.x.x) + dst
                     │  SSL Bump (same SASE-PoC-CA)
                     │  ICAP RESPMOD → ClamAV  (pop01:1344)
                     │  ICAP REQMOD  → Python DLP (mgmt01:1345)
                     │
                     └──► ens3 ──► pop01 ──► internet
```

> **Note (Option D is designed, not tested).** This design is architecturally grounded in Addendum F v3 but has not been empirically validated. The diagnostics (Option C fails) *were* tested on the parallel stack; the TPROXY tests 5 and 10 (source-IP preservation and symmetric return path) were not run because Session 9 was deprioritised. See [Testing: transparent proxy and enforcement](../testing/transparent-proxy-tests.md) (T-TP5/T-TP6).

### Why this is symmetric

The Squid on linuxpop01 is both the interception point and the connection endpoint. Interception happens in `mangle PREROUTING`, before the forwarding decision. The packet is steered to the local socket and never leaves the forwarding pipeline. There is no second stateful router between client and proxy that could break flow symmetry.

### Source-IP preservation via IP_TRANSPARENT

TPROXY marks the packet and the kernel routes it to a local socket with the `IP_TRANSPARENT` socket option. Squid's tproxy listener receives the connection with both src and dst intact. That lets Squid log the real client overlay IP, the prerequisite for the [Identity Bridge](identity-bridge.md) (`external_acl_type` with `%SRC` → NetBird overlay IP → Entra ID user).

### Two complementary Squids

| | pop01 Squid | linuxpop01 Squid |
|---|---|---|
| Mode | explicit (WPAD/PAC, `:3128`) | transparent (TPROXY, `:3129`/`:3130`) |
| Target | clients that respect the proxy | enforcement for all other traffic |
| SSL Bump | yes (same CA) | yes (same CA) |
| ICAP | ClamAV local + DLP mgmt01 | ClamAV pop01 + DLP mgmt01 (over network) |
| Identity Bridge | optional | **primary** (sees the real src IP) |

WPAD/PAC and TPROXY are complementary, not competing: WPAD is the configuration mechanism for compliant traffic, TPROXY is the enforcement boundary for everything else.

---

## The managed-device pivot

*Source: SD-WAN Nulmeting v1.0 §2.1 (3 June 2026); V41 (Intune enforcement session).*

The non-browser-apps gap was the only thing that required transparent interception, and only for **unmanaged** clients at that. With mobile01 now Entra-joined + Intune-enrolled (V40) and the logged-in user not a local admin, endpoint enforcement via MDM is available without the user being able to undo it. This is exactly the Zscaler ZCC and Netskope-client pattern.

V41 delivered the evidence that managed-device enforcement works:

| Intune profile | What it enforces | Evidence |
|---|---|---|
| `2ITCSC1A-SASE-PoC-CA-Root` | SASE-PoC-CA in the Computer Root store | Replaces the manual `certutil` step |
| `2ITCSC1A-SASE-Proxy-PAC` | System-wide proxy via OMA-URI | Overlay `%SRC` in pop01 access.log (B1) |
| `2ITCSC1A-SASE-FW-BypassBlock` | QUIC UDP/443 + DoT TCP/853 blocked | QUIC→TCP fallback + bump proven (B3) |
| `2ITCSC1A-Route-Remediation` | Teams Optimize via the physical gateway | `Find-NetRoute` → physical gw (B2) |

**Consequence:** TPROXY/linuxpop01 is no longer a prerequisite for the SD-WAN pillar, nor for managed devices. linuxpop01 returns to its original role: defense-in-depth for unmanaged devices. Session 9 (TPROXY implementation) was deprioritised in favour of the rubric-bearing sessions and was not executed within the project period. The architecture is fully worked out and validated on the `.11` environment; the implementation specification (Addendum F v3) is ready for possible future execution. See [Decision: Managed Windows Devices Scope](../decisions/managed-devices-scope.md).

---

## Proxy bypass mitigations

*Source: `SASE_PoC_Proxy_Bypass_Mitigaties.md` (May 2026).*

The Proxy Bypass document inventories four remaining bypass routes after transparent interception is in place:

| Gap | Attack | Mitigation | Status |
|---|---|---|---|
| **1. QUIC** | HTTP/3 over UDP/443 passes the TCP interception | `iptables -A FORWARD -i wt0 -p udp --dport 443 -j DROP` on linuxpop01; the browser falls back to TCP/443 | Covered on the sandbox by the Intune firewall CSP (V41) |
| **2. Non-standard ports** | SSH tunnel on 22, SOCKS5 on 1080, etc. | Deny-all FORWARD + explicit whitelist (80, 443, 53, 123, ICMP) | Worked out architecturally; production approach is full default-deny |
| **3. DoH/DoT** | DNS over HTTPS/TLS bypasses Unbound RPZ | DoT: TCP/853 blocked (deny-all or an explicit rule); DoH: a Squid ACL blocks known resolvers on SNI | Covered on the sandbox by the Intune firewall CSP (V41) |
| **4. VPN-in-VPN** | A second WireGuard/OpenVPN client on the device | Three-gate circular dependency: CA compliance → NetBird auth → Intune firewall rules. Architectural boundary: defended, not solved. | Documented as a known limitation |

Gaps 1-3 move from pf rules on pop01 to iptables FORWARD on linuxpop01 once TPROXY is implemented. That incidentally resolves finding 01.11: the floating-rule/`quick` interaction that broke the pf rdr on OPNsense does not exist in iptables.

---

## Rubric relevance

The transparent proxy and its research touch several rubric criteria, even without a live implementation on the sandbox:

| Rubric criterion | Contribution |
|---|---|
| **SWG, Traffic routing** | The research shows architecturally how "fully forced through the SWG" is reached for non-browser traffic. The managed-device variant (Intune) is proven live (V41). |
| **SWG, Identity-based access** | TPROXY preserves the client overlay IP (`%SRC`), the prerequisite for the Identity Bridge (overlay IP → Entra ID user → persona ACL). Without source-IP preservation, identity-based filtering is impossible for transparently intercepted traffic. This was the core motivation for the whole research. |
| **SWG, Malware inspection** | Option D keeps the full ICAP pipeline (ClamAV + DLP) for transparently intercepted traffic. |
| **General, Troubleshooting** | The `SYN_RCVD` diagnostics (TransProxy_Verslag02) is a deep analysis of an asymmetric return path, exactly the kind of root-cause analysis that "In-depth analysis" demands. |
| **General, Architecture** | Five options analysed systematically, three implemented and tested, a wrong diagnosis corrected, the architecture decision revised on the basis of evidence. |
| **General, Open-source tool choice** | Weighing pf (FreeBSD) against iptables/TPROXY (Linux) on kernel-specific limitations, not on preference. |
| **General, Collaboration** | The teammate's finding and decision doc were the starting point; the diagnostics built on his implementation; his Option B was validated architecturally. |

---

## Known issues / gotchas

**`pf rdr` does not work on `wt0`.** This is the original limitation (V17, plugins#3857) that set the whole research in motion. WireGuard-forwarded traffic is injected straight into the kernel routing stack after decapsulation and appears as egress, not ingress. See [Finding: wt0 pf rdr limitation](../findings/wt0-pf-rdr-limitation.md).

**"Add Exit Node" forces masquerade ON.** NetBird's exit-node feature applies masquerade in nftables (`netbird-rt-postrouting`) without a UI toggle. Use a Network Route instead (`Add Route`, range `0.0.0.0/0`, masquerade OFF) to preserve the source IP.

**rdr to 127.0.0.1 fails on FreeBSD.** Squid's NAT lookup cannot retrieve the original destination on a rdr to loopback for forwarded traffic. The error: `NAT lookup failed to locate original IPs`. Redirecting to the WAN IP resolves that but introduces the return-path problem.

**iptables FORWARD ordering.** libvirt (on the GNS3 host) injects a `REJECT` rule into the FORWARD chain. Always use `-I FORWARD 1` instead of `-A FORWARD`. See [Finding: iptables FORWARD ordering](../findings/iptables-forward-ordering.md).

**TPROXY routing persistence.** `ip rule add fwmark 1 lookup 100` and `ip route add local 0.0.0.0/0 dev lo table 100` do not survive a reboot. Persist them via a systemd unit or a netplan hook.

---

## Related

- [Squid: explicit proxy](squid.md): the complementary proxy mode on pop01
- [Identity Bridge](identity-bridge.md): requires the real client overlay IP that TPROXY preserves
- [Decision: WPAD/PAC vs Transparent Proxy](../decisions/wpad-vs-transparent-proxy.md): the original choice (V17/V19)
- [Decision: Managed Windows Devices Scope](../decisions/managed-devices-scope.md): the pivot to endpoint enforcement
- [Finding: wt0 pf rdr limitation](../findings/wt0-pf-rdr-limitation.md): the trigger for the research
- [Finding: iptables FORWARD ordering](../findings/iptables-forward-ordering.md): relevant to the TPROXY iptables configuration
- [Concept: SSL Bump](../concepts/ssl-bump.md): shared CA chain between both Squids
- [Testing: transparent proxy and enforcement](../testing/transparent-proxy-tests.md): the diagnostic tests and the managed-device variant
- [Runbook 03: Proxy & WPAD](../runbooks/03-proxy-wpad.md): operational instructions for the explicit proxy

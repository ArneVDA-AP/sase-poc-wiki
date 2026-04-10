---
title: "Finding: wt0 pf rdr limitation"
tags: [finding, network, proxy, squid]
---

# Finding: wt0 pf rdr limitation

**Component:** [Squid](../components/squid.md)  
**Severity:** Blocker

## What happened

Transparent proxy via `pf rdr` was configured to intercept TCP traffic on the `wt0` (NetBird WireGuard) interface and redirect it to Squid. Traffic from mobile01 via the NetBird tunnel was not being intercepted.

`tcpdump -i wt0 -n 'tcp'` showed 0 packets while mobile01 was actively browsing.

## Root cause

WireGuard creates a Layer 3 routed interface. Traffic routed via `wt0` does not appear as ingress frames on that interface from the packet filter's perspective. `pf rdr` operates on ingress — it cannot see and therefore cannot redirect WireGuard-forwarded traffic.

This is a documented, known limitation. OPNsense GitHub issue #3857 — closed as "not planned."

## Resolution / workaround

Switched to explicit proxy mode with WPAD/PAC. The PAC file at `http://wpad.sandbox.local/wpad.dat` instructs the browser to send traffic directly to `PROXY 100.70.154.79:3128`. No `pf rdr` rule is needed.

See [Decision: WPAD/PAC vs transparent proxy](../decisions/wpad-vs-transparent-proxy.md).

## Lessons

- WireGuard (Layer 3 VPN) is fundamentally incompatible with `pf rdr` transparent proxy
- For any ZTNA overlay-based architecture, explicit proxy with WPAD/PAC is the correct pattern
- `tcpdump` with TCP filter on the suspected interface is the right diagnostic: 0 packets means the interface is not where the traffic appears

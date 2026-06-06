---
title: "Decision: WPAD/PAC vs Transparent Proxy"
tags: [decision, proxy, wpad, squid, sase, network]
---

# Decision: WPAD/PAC vs Transparent Proxy

**Status:** Implemented  
**Date:** March 2026 (Verslag19)

## Context

Clients on the NetBird overlay need all their HTTP/HTTPS traffic routed through Squid for inspection. Two approaches exist: transparent interception (firewall silently redirects port 80/443) and explicit proxy via WPAD/PAC (client is configured to send traffic to the proxy address).

The handbook specified transparent proxy via `pf rdr`.

## Options considered

| Option | Pro | Con |
|--------|-----|-----|
| **Transparent proxy (pf rdr)** | No client configuration required | Does not work on WireGuard (`wt0`) interface — `pf rdr` does not see ingress on Layer 3 VPN interfaces |
| **WPAD/PAC explicit proxy** | Works regardless of underlay network; correct SASE pattern | Requires client to read proxy setting; non-browser apps may bypass |

## Decision

WPAD/PAC explicit proxy, with the PAC file served by Caddy on mgmt01 at `http://wpad.sandbox.local/wpad.dat`.

Transparent interception via `pf rdr` is not viable *on pop01's `wt0` interface*. WireGuard is a Layer 3 routed interface; `pf rdr` operates on ingress frames and does not see traffic forwarded through `wt0`. Confirmed via `tcpdump -i wt0 -n 'tcp'` showing 0 packets (Verslag17, Bevinding 17.10/17.11; OPNsense GitHub issue #3857, closed as "not planned"). That limitation is specific to redirecting on the firewall behind the tunnel; it does not rule out transparent interception elsewhere in the path.

WPAD/PAC is also the architecturally correct pattern for a SASE stack: the commercial equivalent (Zscaler Client Connector) distributes proxy configuration to the browser in exactly this way.

WPAD/PAC stays the **primary** routing mechanism. It does leave a gap: non-browser apps that ignore the system proxy bypass Squid. The follow-on transparent-proxy research closed that gap differently — not with `pf rdr` on `wt0`, but with a **post-tunnel TPROXY on linuxpop01** (the exit node), which intercepts the decapsulated traffic *after* the WireGuard tunnel rather than on the firewall behind it. That sidesteps the `wt0` limitation entirely and captures all traffic regardless of client configuration. The asymmetric-return-path diagnostics behind that approach are validated on the parallel stack; the TPROXY design itself (Option D) is designed but not yet integrated into the sandbox. So WPAD/PAC and transparent proxy are **complementary**: WPAD/PAC configures compliant traffic, transparent proxy is the all-traffic enforcement boundary. See [Component: Transparent Proxy](../components/transparent-proxy.md).

## Consequences

- mobile01 must be configured with the PAC file URL (manually via Windows Settings → Proxy → "Use setup script")
- Non-browser applications that ignore the system proxy setting bypass Squid (partially compensated by Suricata on vtnet0; fully addressed by the complementary [transparent proxy on linuxpop01](../components/transparent-proxy.md), which is designed but not yet integrated into the sandbox)
- Squid must listen on the NetBird overlay IP (`100.70.154.79:3128`) via pre-auth include, not the GUI
- Caddy on mgmt01 serves the PAC file; the `sandbox.local` NetBird Custom DNS Zone routes the `wpad.sandbox.local` query to pop01 Unbound (the NetBird primary nameserver), which resolves it to the Caddy host on mgmt01

See also: [Component: Transparent Proxy](../components/transparent-proxy.md), [Finding: wt0 pf rdr limitation](../findings/wt0-pf-rdr-limitation.md), [Finding: pre-auth ssl-bump params](../findings/pre-auth-ssl-bump-params.md)

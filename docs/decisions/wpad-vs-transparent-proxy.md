---
title: "Decision: WPAD/PAC vs Transparent Proxy"
tags: [decision, proxy, wpad, squid, sase, network]
---

# Decision: WPAD/PAC vs Transparent Proxy

**Status:** Implemented  
**Date:** March 2026 (Verslag19)

## Context

BYOD clients on the NetBird overlay need all their HTTP/HTTPS traffic routed through Squid for inspection. Two approaches exist: transparent interception (firewall silently redirects port 80/443) and explicit proxy via WPAD/PAC (client is configured to send traffic to the proxy address).

The handbook specified transparent proxy via `pf rdr`.

## Options considered

| Option | Pro | Con |
|--------|-----|-----|
| **Transparent proxy (pf rdr)** | No client configuration required | Does not work on WireGuard (`wt0`) interface — `pf rdr` does not see ingress on Layer 3 VPN interfaces |
| **WPAD/PAC explicit proxy** | Works regardless of underlay network; correct SASE pattern | Requires client to read proxy setting; non-browser apps may bypass |

## Decision

WPAD/PAC explicit proxy, with the PAC file served by Caddy on mgmt01 at `http://wpad.sandbox.local/wpad.dat`.

The transparent proxy option was not a viable alternative — it is technically impossible on the `wt0` interface. WireGuard is a Layer 3 routed interface; `pf rdr` operates on ingress frames and does not see traffic forwarded through `wt0`. Confirmed via `tcpdump -i wt0 -n 'tcp'` showing 0 packets (Verslag17, Bevinding 17.10/17.11; OPNsense GitHub issue #3857, closed as "not planned").

WPAD/PAC is also the architecturally correct pattern for a SASE stack: the commercial equivalent (Zscaler Client Connector) distributes proxy configuration to the browser in exactly this way.

## Consequences

- mobile01 must be configured with the PAC file URL (manually via Windows Settings → Proxy → "Use setup script")
- Non-browser applications that ignore the system proxy setting bypass Squid (partially compensated by Suricata on vtnet0)
- Squid must listen on the NetBird overlay IP (`100.70.154.79:3128`) via pre-auth include, not the GUI
- Caddy on mgmt01 serves the PAC file; the `wpad.sandbox.local` DNS name resolves via NetBird Custom DNS Zone

See also: [Finding: wt0 pf rdr limitation](../findings/wt0-pf-rdr-limitation.md), [Finding: pre-auth ssl-bump params](../findings/pre-auth-ssl-bump-params.md)

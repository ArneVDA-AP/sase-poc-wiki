---
title: "Concept: WPAD/PAC — Web Proxy Auto-Configuration"
tags: [proxy, wpad, sase, network, squid]
---

# Concept: WPAD/PAC — Web Proxy Auto-Configuration

**One-line definition:** A two-part standard where WPAD (Web Proxy Auto-Discovery) lets browsers discover a proxy server via DNS, and PAC (Proxy Auto-Configuration) is the JavaScript file the browser executes to decide per-URL whether to use the proxy or connect directly.

## How it applies here

WPAD/PAC is the mechanism that routes every browser request on mobile01 through Squid on pop01 — without hardcoding a proxy address in each browser. The discovery chain:

1. mobile01's DNS search domain is `sandbox.local` (distributed by NetBird DNS configuration)
2. Browser queries `wpad.sandbox.local` → resolves to mgmt01 overlay IP `100.70.135.241` (via NetBird Custom DNS Zone)
3. Browser fetches `http://wpad.sandbox.local/wpad.dat`
4. Browser executes `FindProxyForURL()` for each URL → returns `PROXY 100.70.154.79:3128` (Squid)
5. All non-internal traffic flows through Squid for inspection

**Why WPAD/PAC and not transparent proxy:** Transparent proxy via `pf rdr` cannot intercept WireGuard-routed traffic on the `wt0` interface — WireGuard is Layer 3 and `pf rdr` does not see ingress on that interface from the kernel's perspective. WPAD/PAC explicit proxy is not a workaround; it is the correct pattern for overlay-based BYOD. See [Decision: WPAD/PAC vs transparent proxy](../decisions/wpad-vs-transparent-proxy.md).

**Explicit vs transparent:**
- Explicit proxy: client knows the proxy address and sends requests directly to it (HTTP `GET http://...` or HTTPS `CONNECT` then the request)
- Transparent proxy: firewall silently redirects port 80/443 to the proxy; client is unaware

With explicit proxy mode, Squid can also handle HTTPS via `CONNECT` tunnel (and SSL Bump to inspect the content). Transparent HTTPS requires more complex `pf divert` rules that also do not work on `wt0`.

## Where it appears in the stack

**[Caddy](../components/caddy.md)** — serves the PAC file at `http://wpad.sandbox.local/wpad.dat` with the correct MIME type (`application/x-ns-proxy-autoconfig`). Runs on mgmt01's overlay IP (`100.70.135.241`).

**[Squid](../components/squid.md)** — the proxy that the PAC file points to: `PROXY 100.70.154.79:3128`. Squid runs the pre-auth include listener on this NetBird overlay IP with full ssl-bump parameters.

**[NetBird](../components/netbird.md)** — provides two things WPAD depends on: (1) the overlay network that makes mgmt01's IP reachable from mobile01, and (2) the Custom DNS Zone that resolves `wpad.sandbox.local` to mgmt01's overlay IP.

## Key distinctions

**PAC file must return DIRECT for overlay IPs** — if the PAC file returned `PROXY ...` for `100.64.0.0/10` (the NetBird overlay subnet), browser requests to the NetBird management console would be routed through Squid, which would then try to fetch via the tunnel — a routing loop. The PAC file must explicitly exclude overlay and internal subnets.

**WPAD only covers browser HTTP/HTTPS** — applications that do not read the system proxy setting (many non-browser apps, command-line tools) bypass Squid entirely. Suricata's network-layer inspection on vtnet0 partially compensates by detecting anomalies in direct connections.

**mobile01 is configured manually** — Windows Settings → Proxy → "Use setup script" → URL: `http://wpad.sandbox.local/wpad.dat`. In a production deployment, this would be distributed via Group Policy or MDM.

## Sources

- `raw/Doc1_Squid_WPAD_PAC.md` §1–2 (WPAD/PAC architecture, PAC file, why not transparent proxy)
- `raw/Verslag19.md` (WPAD/PAC implementation session)

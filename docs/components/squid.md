---
title: "Squid — Explicit Proxy, WPAD/PAC, SSL Bump, URL Filtering"
tags: [squid, opnsense, proxy, wpad, pac, ssl-bump, tls, icap, sase, caddy, netbird]
---

# Squid — Explicit Proxy, WPAD/PAC, SSL Bump, URL Filtering

**Role:** The SWG ingress point — every HTTP/HTTPS transaction from a BYOD client passes through Squid before reaching the internet. Squid performs URL filtering, HTTPS decryption (SSL Bump), and orchestrates the ICAP inspection pipeline.  
**Version:** Squid 6.x (bundled with OPNsense 25.1)  
**Config location:** `/usr/local/etc/squid/squid.conf` (OPNsense-generated) + `/usr/local/etc/squid/pre-auth/*.conf` (persistent custom directives)

---

## How it works in this stack

Squid runs in **explicit proxy mode**: clients are configured via a PAC file to send their HTTP/HTTPS traffic directly to Squid on `100.70.154.79:3128` (pop01's NetBird overlay IP). This is the opposite of a transparent proxy, where the firewall silently intercepts traffic.

The choice of explicit mode is not optional — transparent proxy via `pf rdr` cannot intercept WireGuard-routed traffic on the `wt0` interface. See [Finding: wt0 pf rdr limitation](../findings/wt0-pf-rdr-limitation.md) and [Decision: WPAD/PAC vs Transparent Proxy](../decisions/wpad-vs-transparent-proxy.md).

**WPAD/PAC distribution:** The PAC file is hosted by [Caddy](caddy.md) on mgmt01 at `http://wpad.sandbox.local/wpad.dat`. Clients discover this via a NetBird Custom DNS Zone that resolves `wpad.sandbox.local` to mgmt01's overlay IP. Mobile01 is configured via Windows Settings → Proxy → "Use setup script".

**SSL Bump:** Squid decrypts HTTPS traffic using a self-signed CA (`SASE-PoC-CA`, installed on client trust stores). This is what enables the ICAP pipeline to inspect HTTPS content. Without SSL Bump, ClamAV and the Python DLP server see only encrypted bytes.

**URL Filtering:** Two blacklist mechanisms run before any `allow` rule (first-match-wins):
1. Manual blacklist: `gambling.com`, `.bet365.com`, `.pokerstars.com`
2. UT1 Toulouse Remote ACL (`dsi.ut-capitole.fr/blacklists`) — categories: adult, malware, phishing, gambling

**ICAP orchestration:** Squid routes traffic to two ICAP services:
- REQMOD → Python DLP server on `mgmt01:1345` (POST/PUT/PATCH uploads)
- RESPMOD → ClamAV/c-icap on `127.0.0.1:1344` (all responses/downloads)

---

## Configuration

### Squid listeners

The OPNsense GUI generates one listener on the LAN IP. A second listener on the NetBird overlay IP is added via the pre-auth include mechanism (a documented Squid feature, not a hack):

```bash
echo 'http_port 100.70.154.79:3128 ssl-bump cert=/var/squid/ssl/ca.pem \
  dynamic_cert_mem_cache_size=10MB generate-host-certificates=on' \
  > /usr/local/etc/squid/pre-auth/netbird-listener.conf
configctl proxy restart
```

The `ssl-bump` parameters must be included in this directive. The GUI only adds ssl-bump to its own generated listener. Without them, traffic via the PAC file tunnels through as plain CONNECT without inspection. See [Finding: pre-auth ssl-bump params](../findings/pre-auth-ssl-bump-params.md).

### Allowed subnets

`100.64.0.0/10` (NetBird CGNAT range) must be added to Squid ACL via GUI → Forward Proxy → Access Control List → Allowed Subnets. Without this, all NetBird clients receive `TCP_DENIED/403`.

### No-bump list

Sites excluded from SSL inspection (configured via GUI):
- `login.microsoftonline.com` — **required**: protects the Entra ID OIDC flow. If Squid bumps the Microsoft login page, NetBird authentication fails for all clients.
- `.microsoft.com`, `.paypal.com`, `.apple.com`, `.banking.example.com` — demonstrative entries.

### ACL order in squid.conf

```squid
http_access deny blackList
http_access deny remoteblacklist_UT1
http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports
http_access allow localnet
http_access allow subnets
http_access deny all
```

The deny rules for blacklists appear before all allow rules. Squid is first-match-wins.

---

## Integration points

| Component | Direction | What |
|-----------|-----------|------|
| [Caddy](caddy.md) on mgmt01 | → client | Serves PAC file; client uses it to find Squid |
| [NetBird](netbird.md) | dependency | wt0 overlay IP `100.70.154.79` must exist; NetBird DNS zone provides `wpad.sandbox.local` resolution |
| [ClamAV/c-icap](clamav-cicap.md) | ICAP RESPMOD | Squid sends all responses to c-icap on `127.0.0.1:1344` for malware + DLP scan |
| [Python DLP](python-dlp.md) | ICAP REQMOD | Squid sends POST/PUT/PATCH requests to mgmt01:1345 for upload DLP |
| [Suricata](suricata.md) | parallel | Suricata on vtnet0 sees Squid's re-encrypted upstream connections; they share the same WAN interface |
| Entra ID / OIDC | no-bump | `login.microsoftonline.com` is in the no-bump list so CA evaluation is not disrupted |

---

## Known issues / gotchas

**"Clear log" in WebUI destroys the logfile** — see [Finding: Squid clearlog](../findings/squid-clearlog-destroys-file.md). Use `> /var/log/squid/access.log` to truncate without deletion.

**Squid segfault on stop is cosmetic** — `configctl proxy restart` prints "Segmentation fault" when stopping Squid. The daemon restarts correctly. Zero functional impact. Documented in multiple sessions.

**Browser cache masks URL filtering** — when testing URL blocks, always use `curl.exe -x http://100.70.154.79:3128 http://target.com` rather than a browser. Browser cache can serve a cached response even after the URL is blocked.

**`squid -k parse` segfaults at end** — the parse output before the crash is valid and useful for diagnostics. The segfault is an OPNsense/FreeBSD quirk, not a configuration error.

**StevenBlack hosts format is incompatible** — OPNsense Remote ACLs expect a tar.gz archive in Squid ACL format, not a hosts file. Use UT1 Toulouse instead. See [Finding: StevenBlack incompatible](../findings/stevenblack-incompatible.md).

---

## Related

- [Architecture overview](../overview/architecture.md)
- [Concept: WPAD/PAC](../concepts/wpad-pac.md)
- [Concept: SSL Bump](../concepts/ssl-bump.md)
- [Concept: ICAP](../concepts/icap.md)
- [Concept: DLP](../concepts/dlp.md)
- [Decision: WPAD/PAC vs Transparent Proxy](../decisions/wpad-vs-transparent-proxy.md)
- [Decision: Two-layer DLP](../decisions/two-layer-dlp.md)
- [Finding: wt0 pf rdr limitation](../findings/wt0-pf-rdr-limitation.md)
- [Finding: pre-auth ssl-bump params](../findings/pre-auth-ssl-bump-params.md)
- [Finding: Squid clearlog destroys file](../findings/squid-clearlog-destroys-file.md)
- [Finding: StevenBlack incompatible](../findings/stevenblack-incompatible.md)

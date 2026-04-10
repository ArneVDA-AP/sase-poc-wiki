---
title: "Runbook: Proxy & WPAD"
tags: [runbook, squid, ssl-bump, wpad-pac, caddy, swg]
---

# Runbook: Proxy & WPAD

**Source:** `raw/Doc1_Squid_WPAD_PAC.md`
**Node(s):** pop01 (OPNsense — Squid) + mgmt01 (Caddy — WPAD)
**Prerequisites:** [Runbook 02: ZTNA Overlay](02-ztna-overlay.md) completed
**Status:** Operational

---

## Prerequisites checklist

- [ ] NetBird overlay operational — pop01 has overlay IP `100.70.154.79`
- [ ] mgmt01 has overlay IP `100.70.135.241`
- [ ] Caddy running on mgmt01 (part of NetBird Docker stack)
- [ ] OPNsense 25.1 with Squid enabled on pop01

---

## Step 1: Disable transparent proxy

Via OPNsense GUI → Services → Squid Web Proxy → Forward Proxy → General: disable transparent HTTP proxy. Squid should only have:

```
http_port 10.0.0.1:3128
```

The LAN interface stays configured — Squid requires at least one interface with a valid IP to start.

---

## Step 2: Add NetBird overlay subnet to Squid ACL

Without this, `curl -x http://100.70.154.79:3128 http://example.com` returns `TCP_DENIED/403`.

Via GUI: Forward Proxy → Access Control List → Allowed Subnets → add `100.64.0.0/10`.

This generates:

```squid
acl subnets src 100.64.0.0/10
http_access allow subnets
```

**Verify:** `curl -x http://100.70.154.79:3128 http://example.com` shows `TCP_MISS/200` in access log.

---

## Step 3: Create pre-auth listener on overlay IP

The OPNsense GUI cannot bind Squid to `wt0` (no managed IP). Use the pre-auth include mechanism instead:

```bash
echo 'http_port 100.70.154.79:3128' > /usr/local/etc/squid/pre-auth/netbird-listener.conf
configctl proxy restart
```

> **Gotcha: SSL-Bump parameters MUST be added to this listener later (Step 9).** A bare `http_port` without ssl-bump flags means traffic through this listener passes as a CONNECT tunnel without inspection. This is fixed in Step 9 — do not forget.
> See [Finding: pre-auth ssl-bump params](../findings/pre-auth-ssl-bump-params.md).

**Verify:**

```bash
sockstat -4 -l | grep squid
# Must show TWO listeners:
# squid  squid  ...  tcp4  10.0.0.1:3128      *:*
# squid  squid  ...  tcp4  100.70.154.79:3128  *:*
```

---

## Step 4: Install NetBird client on mgmt01 + trust Caddy CA

mgmt01 needs a NetBird overlay IP so it is reachable for the WPAD DNS zone:

```bash
curl -fsSL https://pkgs.netbird.io/install.sh | sudo sh
sudo netbird up --management-url https://netbird.sandbox.local --setup-key 66E94C03-C0E1-4294-B68B-48261ADD6B22
```

The host-level NetBird daemon does not trust the Caddy internal CA (Docker containers do via volume mount). Fix:

```bash
docker cp netbird-deploy-caddy-1:/data/caddy/pki/authorities/local/root.crt /tmp/caddy-root.crt
sudo cp /tmp/caddy-root.crt /usr/local/share/ca-certificates/caddy-root.crt
sudo update-ca-certificates
sudo netbird service restart
```

> **Gotcha: Service restart is mandatory after CA import.** Without restart the error changes from "context canceled" to "deadline exceeded" — the daemon runs with the old trust store.

**Verify:** `netbird status` shows Connected, overlay IP `100.70.135.241`.

---

## Step 5: Configure Caddy as WPAD server

Add volume mount to the caddy service in `docker-compose.yml`:

```yaml
volumes:
  - netbird_caddy_data:/data
  - ./Caddyfile:/etc/caddy/Caddyfile
  - ./wpad:/srv/wpad:ro
```

Add two server blocks to the Caddyfile — **both are required**:

```caddyfile
http://wpad.sandbox.local {
    header /wpad.dat Content-Type application/x-ns-proxy-autoconfig
    root * /srv/wpad
    file_server
}

wpad.sandbox.local {
    tls internal
    header /wpad.dat Content-Type application/x-ns-proxy-autoconfig
    root * /srv/wpad
    file_server
}
```

> **Gotcha: WinHTTP tries HTTPS even for HTTP PAC URLs.** Windows tries to fetch the PAC file via HTTPS first, regardless of the configured URL scheme. Without the HTTPS server block, Caddy logs show `TLS handshake error`. The second block with `tls internal` resolves this.

Create the PAC file at `~/netbird-deploy/wpad/wpad.dat`:

```javascript
function FindProxyForURL(url, host) {
    if (isPlainHostName(host)) return "DIRECT";
    if (shExpMatch(host, "10.*")) return "DIRECT";
    if (shExpMatch(host, "100.64.*")) return "DIRECT";
    if (shExpMatch(host, "127.*")) return "DIRECT";
    if (shExpMatch(host, "*.sandbox.local")) return "DIRECT";
    return "PROXY 100.70.154.79:3128";
}
```

Use `shExpMatch` instead of `dnsResolve` or `isInNet` — DNS-based PAC functions can timeout in the WinHTTP context.

> **Gotcha: Container recreation required, not restart.**
> See [Finding: Docker volume recreation](../findings/docker-volume-recreation.md).

```bash
cd ~/netbird-deploy && docker compose up -d caddy
```

---

## Step 6: Add NetBird DNS zone for WPAD discovery

NetBird Dashboard → DNS → Custom DNS Zones:

```
Zone:          sandbox.local
Distribution:  All
Search Domain: Enabled
A-record:      wpad → 100.70.135.241 (mgmt01 overlay IP)
```

**Verify:**

```powershell
nslookup wpad.sandbox.local
# Must resolve to 100.70.135.241
```

---

## Step 7: Create ACL policies for WPAD reachability

If you replaced the default all-to-all policy in Runbook 02, verify these policies exist:

- **Admin-Infrastructure** — admins can reach admin infra (already created)
- **Mobile-to-Services** — BYOD clients can reach mgmt01 WPAD + pop01 proxy (already created)
- **Datacenter Access** — add `SASE-Admins` as source so mgmt01 can reach pop01 Unbound for DNS

Without the DNS ACL, `netbird status` on mgmt01 shows "Nameservers: 0/1 Available".

---

## Step 8: Configure Windows proxy on mobile01

```
Windows Settings → Network & Internet → Proxy:
  Automatically detect settings: Off
  Use setup script: On
  Script Address: http://wpad.sandbox.local/wpad.dat
  Manual proxy:   Off
```

The Caddy root CA must already be imported in the Windows trust store (done during NetBird enrollment).

---

## Step 9: Enable SSL Bump

**Create SASE-PoC-CA** via System → Trust → Authorities:
- Method: Create an internal Certificate Authority
- Key Type: RSA, 2048 bit, SHA256
- Lifetime: 365 days
- Organization: SASE PoC

**Enable SSL inspection** in Forward Proxy settings with the new CA.

**Update the pre-auth include with full ssl-bump parameters:**

```bash
echo 'http_port 100.70.154.79:3128 ssl-bump cert=/var/squid/ssl/ca.pem dynamic_cert_mem_cache_size=10MB generate-host-certificates=on' > /usr/local/etc/squid/pre-auth/netbird-listener.conf
configctl proxy restart
```

**Verify:** Browser shows certificate with issuer `Organization (O): SASE PoC`.

**Configure the no-bump list** (GUI → Forward Proxy → SSL inspection):

```
login.microsoftonline.com
.microsoft.com
.paypal.com
.apple.com
.banking.example.com
```

> **Gotcha: `login.microsoftonline.com` MUST be in the no-bump list.** If Squid bumps the Microsoft login page, the OIDC certificate chain breaks and Entra ID authentication fails for all NetBird clients.

---

## Step 10: Enable URL filtering with UT1 Toulouse

Via GUI → Forward Proxy → Remote Access Control List:

```
Filename:        UT1
URL:             https://dsi.ut-capitole.fr/blacklists/download/blacklists.tar.gz
SSL ignore cert: checked
Categories:      adult, malware, phishing, gambling
```

Manual blacklist via Forward Proxy → Access Control List → Blacklist:

```
gambling.com
.bet365.com
.pokerstars.com
```

**Verify** — use curl, not a browser (browser cache masks blocking):

```powershell
curl.exe -x http://100.70.154.79:3128 http://gambling.com -v
# Expected: HTTP/1.1 403 Forbidden, X-Squid-Error: ERR_ACCESS_DENIED 0
```

---

## Known cosmetic issues

**OPNsense "Clear log" deletes the log file** — the WebUI button deletes rather than truncates. Squid stops logging. Fix:

```bash
> /var/log/squid/access.log   # truncate without deleting
```

See [Finding: Squid clearlog destroys file](../findings/squid-clearlog-destroys-file.md).

**Squid segfault on stop is cosmetic** — occurs during `configctl proxy restart` stop phase on FreeBSD. The daemon restarts correctly. No functional impact.

---

## Final verification

- [ ] `sockstat -4 -l | grep squid` shows both listeners (10.0.0.1 and 100.70.154.79)
- [ ] `curl -x http://100.70.154.79:3128 http://example.com` returns 200
- [ ] `nslookup wpad.sandbox.local` resolves to `100.70.135.241`
- [ ] Browser on mobile01 auto-configures proxy via PAC
- [ ] HTTPS sites show certificate issuer "SASE PoC"
- [ ] `login.microsoftonline.com` shows its own certificate (not bumped)
- [ ] `curl.exe -x http://100.70.154.79:3128 http://gambling.com` returns 403

---

## Related

- [Component: Squid](../components/squid.md)
- [Component: Caddy](../components/caddy.md)
- [Concept: SSL Bump](../concepts/ssl-bump.md)
- [Concept: WPAD/PAC](../concepts/wpad-pac.md)
- [Decision: WPAD/PAC vs Transparent Proxy](../decisions/wpad-vs-transparent-proxy.md)
- [Finding: wt0 pf rdr limitation](../findings/wt0-pf-rdr-limitation.md)
- [Finding: pre-auth ssl-bump params](../findings/pre-auth-ssl-bump-params.md)
- [Finding: StevenBlack incompatible](../findings/stevenblack-incompatible.md)
- [Finding: Squid clearlog destroys file](../findings/squid-clearlog-destroys-file.md)
- [Finding: Docker volume recreation](../findings/docker-volume-recreation.md)

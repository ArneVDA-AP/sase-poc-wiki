---
title: "Finding: pre-auth ssl-bump params required"
tags: [finding, squid, ssl-bump, proxy]
---

# Finding: pre-auth ssl-bump params required

**Component:** [Squid](../components/squid.md)  
**Severity:** Blocker

## What happened

After adding a Squid listener on the NetBird overlay IP via pre-auth include (`http_port 100.70.154.79:3128`), HTTPS traffic from mobile01 via the proxy was not being intercepted by ClamAV or the DLP server. The traffic appeared to pass through without inspection.

## Root cause

The pre-auth include file only contained the bare `http_port` directive without ssl-bump parameters. Without `ssl-bump cert=... generate-host-certificates=on`, Squid handles HTTPS as an opaque `CONNECT` tunnel — it forwards encrypted bytes without decrypting. ICAP services never receive content to inspect.

The OPNsense GUI adds ssl-bump parameters only to the listener it generates (the LAN IP). It has no knowledge of the pre-auth include file.

## Resolution / workaround

The pre-auth include file must contain the full ssl-bump directive:

```bash
echo 'http_port 100.70.154.79:3128 ssl-bump cert=/var/squid/ssl/ca.pem \
  dynamic_cert_mem_cache_size=10MB generate-host-certificates=on' \
  > /usr/local/etc/squid/pre-auth/netbird-listener.conf
configctl proxy restart
```

## Lessons

- Every `http_port` directive that needs SSL Bump must explicitly include the ssl-bump parameters — there is no inheritance from other listeners
- The pre-auth include file IS visible in `squid -k parse` output (both `http_port` lines appear). It is only invisible in `grep http_port squid.conf` because it lives in the `pre-auth/` include directory, not in `squid.conf` itself
- After any change to the pre-auth include, `configctl proxy restart` is required — not a GUI Apply

---
title: "Concept: SSL Bump — HTTPS Interception"
tags: [ssl-bump, tls, squid, icap, proxy, sase]
---

# Concept: SSL Bump — HTTPS Interception

**One-line definition:** Squid's mechanism for decrypting HTTPS traffic by acting as a TLS man-in-the-middle — it re-signs server certificates on the fly using a local CA trusted by clients, enabling content inspection of otherwise opaque encrypted traffic.

## How it applies here

Without SSL Bump, ClamAV and the Python DLP server see only an opaque encrypted stream — useless for malware scanning or DLP. SSL Bump makes HTTPS inspection possible: Squid decrypts each HTTPS connection, passes the cleartext body to the ICAP inspection chain, then re-encrypts toward the origin server.

The SSL Bump CA for this project is `SASE-PoC-CA`, a self-signed root CA installed in the Windows trust store on mobile01. Without this certificate, every HTTPS site appears as an untrusted site to the browser.

**TLS flow with SSL Bump:**

```
mobile01 (browser) ←→ Squid pop01 ←→ Internet (origin server)
  [SASE-PoC-CA cert]      ↕ cleartext          ↕ real TLS
                     ICAP inspection
                 (ClamAV + Python DLP)
```

Squid generates a per-site certificate signed by `SASE-PoC-CA`, substituting its own cert for the origin's real cert. The browser sees a valid cert (trusted via the installed CA) and does not detect the interception.

## Where it appears in the stack

**[Squid](../components/squid.md)** — performs SSL Bump on both listeners. The LAN listener gets ssl-bump parameters from the OPNsense GUI. The NetBird overlay listener (`100.70.154.79:3128`) requires ssl-bump parameters in the pre-auth include file — without them, traffic tunnels through as CONNECT without inspection:

```bash
echo 'http_port 100.70.154.79:3128 ssl-bump cert=/var/squid/ssl/ca.pem \
  dynamic_cert_mem_cache_size=10MB generate-host-certificates=on' \
  > /usr/local/etc/squid/pre-auth/netbird-listener.conf
```

**No-bump list** — sites excluded from SSL inspection:
- `login.microsoftonline.com` — **required**: Squid bumping the Microsoft login page breaks the Entra ID OIDC flow used by NetBird. All client authentication would fail.
- `.microsoft.com`, `.paypal.com`, `.apple.com` — demonstrative exemptions

**[ClamAV/c-icap](../components/clamav-cicap.md)** and **[Python DLP](../components/python-dlp.md)** — depend on SSL Bump to receive cleartext bodies. Without it, they cannot inspect HTTPS content.

## Key distinctions

**ssl-bump on pre-auth listener** — the critical finding documented in [Finding: pre-auth ssl-bump params](../findings/pre-auth-ssl-bump-params.md): if the pre-auth include only has `http_port 100.70.154.79:3128` (no ssl-bump parameters), HTTPS traffic from NetBird clients tunnels as CONNECT without inspection. The full ssl-bump directive must be in the pre-auth conf.

**`--ssl-no-revoke` for Windows curl.exe testing** — curl.exe on Windows uses the schannel TLS stack which attempts CRL/OCSP verification on the SSL Bump certificate. Self-signed CAs have no revocation endpoint, so this fails. Add `--ssl-no-revoke` to all Windows CLI test commands going through the proxy.

**SSL Bump does not apply to no-bump sites** — traffic to `login.microsoftonline.com` passes through as a CONNECT tunnel without decryption, exactly as it would without SSL Bump configured.

## Sources

- `raw/Doc1_Squid_WPAD_PAC.md` §2 (SSL Bump configuration, no-bump list)
- `raw/Verslag20.md` (ssl-bump finding for pre-auth include)
- `raw/Doc2_ClamAV_DLP_Pipeline.md` (dependency on SSL Bump for inspection)

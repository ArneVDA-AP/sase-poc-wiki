---
title: "Finding: curl --ssl-no-revoke Required on Windows for Proxy Tests"
tags: [swg, ssl-bump, finding, workaround]
---

# Finding: curl --ssl-no-revoke Required on Windows for Proxy Tests

**Component:** [Squid](../components/squid.md), [ClamAV/c-icap](../components/clamav-cicap.md)  
**Severity:** Gotcha

## What happened

When running ICAP pipeline tests from mobile01 (Windows 11) using `curl.exe` through the Squid proxy to an HTTPS destination, curl fails with a TLS error unless `--ssl-no-revoke` is specified:

```powershell
# Fails without --ssl-no-revoke
curl.exe -x http://100.70.154.79:3128 -o eicar_test.txt https://secure.eicar.org/eicar.com

# Works
curl.exe -x http://100.70.154.79:3128 --ssl-no-revoke -o eicar_test.txt https://secure.eicar.org/eicar.com
```

Without `--ssl-no-revoke`, the request never reaches the ICAP pipeline — curl fails at TLS handshake.

## Root cause

Windows `curl.exe` uses the Schannel TLS backend, which attempts CRL (Certificate Revocation List) and OCSP validation on every certificate in the chain. When Squid performs SSL Bump, the certificate presented to the client is issued by the SASE-PoC-CA — a locally generated CA with no published CRL or OCSP endpoint. Schannel treats a missing revocation endpoint as a failure and aborts the connection.

## Resolution

Add `--ssl-no-revoke` to all `curl.exe` commands from Windows hosts that route through the SSL Bump proxy to HTTPS destinations. This flag instructs Schannel to skip revocation checks.

This flag does not affect test validity — the ICAP pipeline blocking or passing behavior is entirely independent of the CRL check that Schannel is performing.

## Lessons

- All Windows `curl.exe` proxy tests against HTTPS destinations require `--ssl-no-revoke` in this environment.
- Browser-based tests are unaffected: browsers handle the SASE-PoC-CA via the installed trusted CA and tolerate missing CRL endpoints more gracefully.
- PowerShell `Invoke-WebRequest` uses the same Schannel backend and has the same behavior — the same workaround applies if Invoke-WebRequest is used for testing.
- In production, the SASE-PoC-CA would be distributed via Group Policy and a CRL endpoint would be configured — making this issue moot.

## Related

- [SSL Bump](../concepts/ssl-bump.md)
- [ClamAV/c-icap](../components/clamav-cicap.md)
- [Squid](../components/squid.md)
- [Testing: Acceptance Tests](../testing/acceptance-tests.md)

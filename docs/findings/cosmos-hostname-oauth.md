---
title: "Finding: Cosmos hostname on IP limits OAuth/SSO"
tags: [finding, cosmos, application-gateway, oidc, mfa, tls]
---

# Finding: Cosmos hostname on IP limits OAuth/SSO

**Component:** [Cosmos](../components/cosmos.md)
**Severity:** Gotcha

## What happened

Cosmos was set up with its hostname as the IP `10.0.0.100` (closed lab, no FQDN). The per-route MFA gate works for Uptime Kuma on `kuma.dc.local`, but the `AuthEnabled` toggle is greyed out and cannot be saved for routes on a `*.dc.local` hostname (Gitea). Cross-domain SSO between the Cosmos admin origin and the application routes does not function.

## Root cause

With the hostname set to an IP, Cosmos builds its OAuth redirect URL as `https://10.0.0.100/cosmos-ui/openid`. Browser cookies are domain-bound: a session cookie set on `10.0.0.100` is not valid on `gitea.dc.local`. So when Cosmos tries to complete the OAuth/SSO flow for a route on a different hostname, the cookie set at the admin origin (`10.0.0.100`) is never presented back on the route's domain (`*.dc.local`), and the auth binding cannot be established. Cosmos surfaces this by disabling (greying) the `AuthEnabled` toggle for `*.dc.local` routes — the feature is unavailable, not just unconfigured.

The underlying constraint is architectural, not a bug: an OAuth/SSO redirect-and-cookie flow needs a single parent domain shared by the issuer and the protected routes. An IP literal cannot be a parent of `*.dc.local`, so the cross-domain cookie can never be valid.

## Resolution / workaround

For the PoC the limitation was accepted deliberately — the core gateway functions (identity gate on `kuma.dc.local`, SmartShield, container isolation) all work, and Gitea is intentionally left without the Cosmos MFA gate anyway (it has its own mature auth). See [Decision: Two-layer ZTNA](../decisions/cosmos-two-layer-ztna.md).

The real fix is to give Cosmos an FQDN under the shared `dc.local` parent so the OAuth issuer and the routes live under one domain:

1. Add `cosmos.dc.local` to dc01 `/etc/hosts`.
2. Change the hostname in Cosmos **Configuration → General** to `cosmos.dc.local`.
3. Restart Cosmos and re-validate the OAuth redirect URLs.

After migration the redirect becomes `https://cosmos.dc.local/cosmos-ui/openid`, a cookie on `cosmos.dc.local` shares the `dc.local` parent with `gitea.dc.local`/`kuma.dc.local`, and the `AuthEnabled` toggle becomes usable on `*.dc.local` routes. This migration is tracked as an open point and is deferred in the current build.

## Lessons

- For any reverse proxy / application gateway that does OAuth/SSO across multiple sub-hostnames, set the gateway's own hostname to an FQDN under the same parent domain as the routes from day one — an IP literal forecloses cross-domain cookies.
- A greyed-out UI toggle can be a signal of a structural limitation, not a missing setting. Here the greyed `AuthEnabled` is Cosmos correctly refusing a config that could never produce a valid cookie.
- The limitation and the per-service architecture choice are independent: Gitea is gate-off by design, so on the parallel stack this finding did not block anything — but it would for any future `*.dc.local` service that does need the Cosmos MFA gate.

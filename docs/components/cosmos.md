---
title: "Cosmos — Identity-Aware Application Gateway"
tags: [cosmos, application-gateway, reverse-proxy, mfa, smartshield, zero-trust, ztna, sase, docker, dns]
---

# Cosmos — Identity-Aware Application Gateway

**Role:** Identity-aware application gateway on dc01 — a reverse proxy that puts a per-application login + MFA gate, SmartShield rate-limiting, and Docker container isolation in front of every internal DC service.
**Version:** Cosmos Server v0.22.10
**Config location:** dc01 (`10.0.0.100`) — `/var/lib/cosmos` (mounted as `/config`); Cosmos admin UI at `https://cosmos.dc.local/cosmos-ui/`
**Status:** ✅ PoC-validated on the parallel stack — sandbox integration pending

> **Sidenote — parallel-stack scope.** Cosmos was built and validated on the parallel `SASE_POC` environment (dc01 `10.0.0.100`), not yet integrated into the main sandbox. Where its scope overlaps the sandbox (DNS, the NetBird overlay), the sandbox is source of truth and Cosmos is additive: it adds an application-admission layer on top, it does not replace anything in the sandbox build.

---

## How it works in this stack

The stack already has three layers below Cosmos: NetBird ZTNA decides which device may enter the DC network, OPNsense + Suricata do L3/L4 filtering and IDS, and Squid + ClamAV inspect outbound content. None of those answer the question *which user may open this specific application*. Cosmos is the fourth layer that does.

Concretely, every DC service (Uptime Kuma, Gitea, future services) is reachable **only** through Cosmos on ports 80/443. The service containers are never bound externally — Kuma's `3001/tcp` and Gitea's `3000/tcp` are not exposed on the host — so the only path in is the reverse proxy. Cosmos intercepts each request to a protected route, runs login + MFA, and only then forwards to the backend container. This is the second of two Zero Trust gates: NetBird is network admission (device-level), Cosmos is application admission (user-level). A device that is on the NetBird overlay but whose Cosmos session has expired cannot open any DC application without re-authenticating with MFA. See [Decision: Two-layer ZTNA](../decisions/cosmos-two-layer-ztna.md).

Cosmos folds five distinct functions into one component:

1. **Reverse proxy with container isolation.** Each service lives in its own Docker network (`cosmos-Uptime-Kuma-default`, `cosmos-Gitea-default`). An attacker who compromises one container cannot move laterally to the other — the networks are isolated and the backend ports are not externally bound.
2. **Identity gate with MFA.** Cosmos verifies user identity via login + TOTP (Microsoft / Google Authenticator) before granting access. The session is time-bound: 48 hours in this PoC. An expired session, a different browser, or an incognito window forces re-authentication with MFA.
3. **SmartShield (per route, v0.22.10).** Per-route rate-limiting, bot detection, IP-based throttling, and anti-DDoS. Uptime Kuma is configured at a 12,000 requests/minute threshold. Because every client must already pass NetBird before reaching Cosmos, SmartShield sees NetBird overlay addresses as the source IP — internet scanners never reach Cosmos at all.
4. **Container management and visibility.** The Servapps panel gives the admin visibility over every Docker container on dc01, including containers not deployed through Cosmos. This is the basis for inbound Shadow IT visibility on dc01 (see [Decision: Two-layer ZTNA](../decisions/cosmos-two-layer-ztna.md)).
5. **Session management as access control.** NetBird gives device-level network access; Cosmos adds user-level, time-bound access on top. Session expiry is itself an access-control event — the device stays on the network, but the application closes until the next MFA.

### Uptime Kuma behind the gate

Uptime Kuma is the stack's monitoring dashboard, deployed through Cosmos **with** the MFA gate enabled, reachable at `https://kuma.dc.local`. The dashboard exposes operationally sensitive state — which services are up, historical uptime across the SASE stack — so it gets the maximum stack: NetBird (ZTNA) + Cosmos MFA + Kuma's own login. It monitors the parallel-stack components by IP (OPNsense `10.0.0.1`, Cosmos `10.0.0.100`, mgmt01 `192.168.122.20`, Gitea via the Cosmos IP). The demo flow is: incognito window → `https://kuma.dc.local` → Cosmos login (not Kuma) → credentials → MFA code → dashboard.

Gitea is deployed through Cosmos **without** the MFA gate — a deliberate per-service choice, since Gitea has its own mature auth (SSH keys, PATs, optional 2FA) and stacking Cosmos MFA on top would be double authentication for the same identity. See [Decision: Two-layer ZTNA](../decisions/cosmos-two-layer-ztna.md) for the per-service rationale.

---

## Configuration

Key choices and the reason behind each:

| Setting | Value | Reason |
|---------|-------|--------|
| Hostname | `10.0.0.100` | Closed lab, no FQDN migration done; this limits OAuth/SSO cross-domain. See gotchas + [Finding: hostname on IP limits OAuth](../findings/cosmos-hostname-oauth.md). |
| HTTPS Mode | `SELFSIGNED` | No Let's Encrypt reachable in a closed lab. |
| Allow insecure via local IP | On | Admin access via IP without a certificate warning. |
| Force Multi-Factor Authentication | On (global) | MFA mandatory for every user. |
| Session timeout | 48 hours | Demo convenience — evaluators and team members avoid re-doing MFA every session. Production would be 1 hour or less. |
| Constellation VPN | Not activated | Paid feature in v0.22.10; NetBird ZTNA replaces it entirely (NetBird advertises `10.0.0.0/8` to all peers, giving network reach to dc01). |

Cosmos runs as a privileged `--network host` container with the Docker socket mounted, so it can bind ports 80/443 on the host and manage the other service containers. The Docker daemon on dc01 is pinned to a custom address pool (`192.168.200.0/20`) and MTU 1400 to avoid collisions with GNS3 segments and encapsulation fragmentation — both are mandatory for the stack to work. The full step-by-step build is in [Runbook 13: Cosmos](../runbooks/13-cosmos.md).

When Uptime Kuma is installed via Cosmos Market, Cosmos generates the route security config automatically:

```json
{
  "SmartShield": { "Enabled": true },
  "AuthEnabled": true,
  "BlockCommonBots": true,
  "ThrottlePerMinute": 12000,
  "cosmos-force-network-secured": "true"
}
```

The `cosmos-force-network-secured` label is what ties application access back to the network layer: a container without a Cosmos route gets no external network path, which is also the mechanism behind inbound Shadow IT containment on dc01.

---

## Integration points

| Component | Direction | What |
|-----------|-----------|------|
| [NetBird](netbird.md) | depends on (network admission) | Cosmos is Gate 2; NetBird is Gate 1. Clients reach `10.0.0.100` only through the NetBird overlay, so Cosmos and `*.dc.local` are unreachable without NetBird. NetBird advertising `10.0.0.0/8` is what makes dc01 routable. |
| Unbound on OPNsense (`10.0.0.1`) | DNS dependency | Host overrides map `cosmos.dc.local`, `kuma.dc.local`, `gitea.dc.local` → `10.0.0.100` so clients can resolve the routes. |
| dc01 `/etc/hosts` | DNS dependency | Cosmos runs in Docker and uses Docker's internal resolver (`127.0.0.11`), which does not know `dc.local`. Each `*.dc.local` route must also be added to dc01's hosts file or the route returns an internal server error. |
| NetBird DNS (nameserver rule) | DNS dependency | A NetBird nameserver rule (`10.0.0.1`, match domain `dc.local`) plus an OPNsense `wt0` firewall rule for port 53 lets overlay clients resolve `*.dc.local`. |
| Uptime Kuma / Gitea backends | proxies to | Reverse-proxied on the host; backend ports `3001`/`3000` are not externally bound — Cosmos is the only ingress. |

---

## Known issues / gotchas

- **Hostname on IP breaks cross-domain OAuth/SSO.** With the hostname set to `10.0.0.100`, Cosmos builds its OAuth redirect as `https://10.0.0.100/cosmos-ui/openid`. Browser cookies are domain-bound, so a cookie set on `10.0.0.100` is invalid on `gitea.dc.local` — the `AuthEnabled` toggle is greyed out for `*.dc.local` routes. This is the single biggest architectural limitation of the install. Full analysis and the FQDN-migration path: [Finding: hostname on IP limits OAuth](../findings/cosmos-hostname-oauth.md).
- **Nginx must be masked, not just stopped.** Ubuntu Server may pull nginx as a dependency; it grabs ports 80/443 that Cosmos needs, and `systemctl stop` is not enough because it restarts after a reboot. Mask it.
- **`dc.local` is invisible to Cosmos' internal resolver.** The most surprising gotcha: a route on `kuma.dc.local` fails with `lookup kuma.dc.local: no such host` until the name is added to dc01 `/etc/hosts`. Add every new `*.dc.local` service to that file immediately.
- **SmartShield has no global toggle in v0.22.10.** It is configured per route, automatically when installing via Cosmos Market — you cannot enable it once for all services.
- **Do not install via Portainer / CasaOS / Unraid templates.** They break the Cosmos configuration; always use the official `docker run` command.
- **Constellation VPN is a paid feature in v0.22.10.** Do not try to activate it; NetBird replaces it.
- **Kuma monitors must use IP addresses, not hostnames.** The Kuma container uses Docker's internal DNS, which does not know `dc.local`; use `https://10.0.0.100` (Cosmos) to monitor services behind it, never the internal backend port.

A wider list of build-time pitfalls (nginx masking, daemon.json, DNS search-domain interference, the `wt0` DNS firewall rule) is captured step-by-step in [Runbook 13: Cosmos](../runbooks/13-cosmos.md).

---

## Related

- [Concept: Application Gateway](../concepts/application-gateway.md)
- [Concept: Zero Trust](../concepts/zero-trust.md)
- [Decision: Two-layer ZTNA (NetBird + Cosmos)](../decisions/cosmos-two-layer-ztna.md)
- [Finding: Cosmos hostname on IP limits OAuth/SSO](../findings/cosmos-hostname-oauth.md)
- [Runbook 13: Cosmos](../runbooks/13-cosmos.md)
- [Component: NetBird](netbird.md)

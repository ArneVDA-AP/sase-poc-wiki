---
title: "Decision: Two-Layer Zero Trust (NetBird + Cosmos)"
tags: [decision, cosmos, application-gateway, mfa, zero-trust, ztna, netbird, sase]
---

# Decision: Two-Layer Zero Trust (NetBird + Cosmos)

**Status:** Implemented (PoC-validated on the parallel stack — sandbox integration pending)
**Date:** May 2026 (Cosmos implementation, dc01)

> **Scope.** This decision was built and validated on the parallel `SASE_POC` environment (dc01 `10.0.0.100`), not yet integrated into the main sandbox. It is additive to the sandbox's existing NetBird ZTNA, not a replacement.

## Context

NetBird ZTNA admits a device to the DC network and lets it reach a resource's IP. But network admission is device-scoped: once a device is on the overlay, anyone using it can attempt `10.0.0.100` and reach whatever its ACLs allow. The internal DC services (Uptime Kuma, Gitea, future services) needed a second control that answers a different question — *which user may open this specific application* — with strong authentication, and ideally with per-application granularity so a sensitive dashboard and a developer tool need not carry the same posture.

The question was whether device-level ZTNA is enough on its own, or whether a separate application-admission layer is warranted, and if so, which technology should provide it.

## Options considered

| Option | Pro | Con |
|--------|-----|-----|
| NetBird only (device admission) | One layer to operate; no extra component | No user-level, per-application gate; a shared/unlocked device on the overlay reaches every service; no MFA at the application |
| Cosmos Constellation VPN (built-in) | Single vendor for network + app | Paid feature in v0.22.10; duplicates NetBird's overlay; would mean two overlays |
| App-native auth only (e.g. Gitea's own 2FA) | No proxy layer; per-app by definition | Inconsistent across services; services without strong auth stay exposed; no central session/MFA gate; no container isolation |
| **NetBird (network) + Cosmos (application)** | Two orthogonal gates; user-level MFA per app; per-service granularity; container isolation + SmartShield as a bonus | Extra component on dc01; Cosmos hostname-on-IP limits cross-domain OAuth |

## Decision

Implement Zero Trust as **two layers**: NetBird for network admission (Gate 1, device-level) and [Cosmos](../components/cosmos.md) as an [application gateway](../concepts/application-gateway.md) for application admission (Gate 2, user-level with MFA).

```
GATE 1 — NetBird:  which device may enter the DC network?   (device-level)
GATE 2 — Cosmos:   which user may open this application?     (user-level, login + MFA via TOTP)
```

Cosmos Constellation VPN is explicitly not used (paid, and it would duplicate NetBird's overlay); NetBird advertises `10.0.0.0/8` to all peers, which is what makes dc01 routable, so Cosmos only needs to gate applications, not transport.

Within Gate 2, posture is set **per service**:

- **Uptime Kuma — MFA gate on.** The monitoring dashboard exposes operationally sensitive state across the whole SASE stack, so it gets the maximum stack: NetBird + Cosmos MFA + Kuma's own login.
- **Gitea — MFA gate off (deliberate).** Gitea has mature native auth (SSH keys, PATs, optional 2FA) and developers log in many times a day; a second Cosmos MFA on the same identity would be redundant. Access control here is NetBird (device) + Gitea's own auth (user). A technical detail reinforces the choice: the `AuthEnabled` toggle is greyed for `*.dc.local` routes because of the hostname-on-IP limitation, but the architectural motivation stands independent of that constraint. See [Finding: hostname on IP limits OAuth](../findings/cosmos-hostname-oauth.md).

## Consequences

- Two gates are orthogonal and complementary: Gate 1 cannot enforce per-application user identity, and Gate 2 cannot exist without the network path Gate 1 provides. Neither substitutes for the other.
- A device on the NetBird overlay whose Cosmos session has expired (48h timeout in this PoC) cannot open any MFA-gated DC application without re-authenticating with MFA. Session expiry becomes an access-control event, not just a UX detail.
- The application-gateway layer brings two further benefits for free: per-route SmartShield rate-limiting, and Docker container isolation (each backend in its own network, ports not externally bound), which limits lateral movement if one backend is compromised.
- Per-service granularity is now a first-class capability: the admin decides the posture per application, demonstrated by the Kuma-on / Gitea-off split.
- The cost is one extra component on dc01 and the hostname-on-IP limitation (greyed `AuthEnabled` on `*.dc.local`, no cross-domain OAuth/SSO). Migrating Cosmos to `cosmos.dc.local` would lift that limit — deferred. See [Finding: hostname on IP limits OAuth](../findings/cosmos-hostname-oauth.md).
- Constellation VPN stays off; if a future need for a Cosmos-native overlay arises, the paid-feature and double-overlay trade-offs would have to be reconsidered.

See also: [Component: Cosmos](../components/cosmos.md), [Component: NetBird](../components/netbird.md), [Concept: Application Gateway](../concepts/application-gateway.md), [Concept: Zero Trust](../concepts/zero-trust.md), [Runbook 13: Cosmos](../runbooks/13-cosmos.md)

---
title: "Concept: Application Gateway"
tags: [application-gateway, cosmos, reverse-proxy, mfa, smartshield, zero-trust, ztna, sase]
---

# Concept: Application Gateway

**One-line definition:** An identity-aware reverse proxy that sits in front of internal applications and gates each one on user identity (login + MFA), session validity, and per-route protections — answering *which user may open this application*, distinct from *which device may enter the network*.

## How it applies here

In this project the application gateway is the fourth and topmost layer of the SASE stack, implemented by [Cosmos](../components/cosmos.md) on dc01. The three layers below it answer different questions: NetBird ZTNA answers *which device may enter the DC network*, OPNsense + Suricata do L3/L4 filtering, and Squid + ClamAV inspect outbound content. None of them decides whether a given **user** may open a given **application**. That is the gap the application gateway fills.

The gateway enforces this by being the only ingress to the backend services. Every DC service (Uptime Kuma, Gitea) is reachable only through Cosmos on ports 80/443; the backend container ports (`3001`, `3000`) are never bound externally. A request to a protected route is intercepted, the user authenticates with login + TOTP, and only then is the request forwarded to the backend. Four properties make this an *application gateway* rather than a plain reverse proxy:

- **Identity gate** — login + MFA per protected route, not just transport.
- **Session management as access control** — the Cosmos session is time-bound (48h in this PoC). When it expires the application closes, even though the device is still on the network.
- **SmartShield** — per-route rate-limiting, bot detection, and anti-DDoS. Because clients arrive over the NetBird overlay, the source IPs SmartShield sees are overlay addresses, so internet scanners never reach it.
- **Container isolation** — each application runs in its own Docker network so a compromise of one backend cannot move laterally to another.

The gateway is the second of two Zero Trust gates: device admission (NetBird) then application admission (Cosmos). See [Decision: Two-layer ZTNA](../decisions/cosmos-two-layer-ztna.md) and [Concept: Zero Trust](zero-trust.md).

## Where it appears in the stack

- **[Cosmos](../components/cosmos.md)** — the sole application gateway in this project. Reverse-proxies Uptime Kuma (`kuma.dc.local`, MFA gate enabled) and Gitea (`gitea.dc.local`, MFA gate deliberately off) on dc01; applies SmartShield per route and isolates each backend in its own Docker network.
- **[NetBird](../components/netbird.md)** — the layer *below* the gateway. It is not an application gateway (it admits devices to the network); it is the precondition that makes the gateway reachable, since `10.0.0.100` and `*.dc.local` are only routable over the overlay.

## Key distinctions

**Application gateway vs SWG (Secure Web Gateway).** Both are proxies, but they face opposite directions. An SWG (Squid in this stack) is an *outbound* gateway: it inspects traffic from internal users going *out* to the internet, applying URL filtering, malware scanning, and DLP. An application gateway is an *inbound* gateway: it controls traffic from users coming *in* to internal applications, gating on identity and MFA. Cosmos is not an SWG and sees none of the outbound traffic Squid handles.

**Application gateway vs ZTNA network-admission.** ZTNA network-admission (NetBird) decides which *device* may join the network and reach a resource's IP; once admitted, the device can attempt any service its ACLs allow. The application gateway adds a second, user-level gate *per application*: a device that is on the NetBird overlay but whose Cosmos session has expired still cannot open any DC application without re-authenticating with MFA. Network admission is device-scoped and one-time per connection; application admission is user-scoped, per-application, and re-checked on session expiry.

**Per-application granularity.** The gateway can apply a different posture per service. Uptime Kuma carries the MFA gate (operationally sensitive monitoring data); Gitea does not (it has its own mature auth, so a second MFA layer would be redundant). This per-service choice is a property of the application-gateway model, not a limitation. See [Decision: Two-layer ZTNA](../decisions/cosmos-two-layer-ztna.md).

## Sources

- `rayan_Cosmos Implementatie SASE PoC.md` §1, §12 (four-layer model, identity gate, SmartShield, deliberate architecture choices)
- `DEMO_SCRIPT_COSMOS_MFA.md` (MFA gate flow, two-gate framing)
- `SASE_PoC_Rayan_State_Snapshot_Juni2026.md` §2 ZTNA (Cosmos as per-application access control, Gate 2)

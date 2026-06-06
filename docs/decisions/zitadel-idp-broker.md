---
title: "Decision: Zitadel as IdP Broker"
tags: [decision, zero-trust, sase, network]
---

# Decision: Zitadel as IdP Broker

**Status:** Implemented  
**Date:** March 2026 (Verslag19, Doc6)

## Context

NetBird requires an OIDC provider for user authentication. The handbook (v4 §19) described a direct Entra ID connection: Entra ID configured as OIDC provider directly in the NetBird Dashboard. The `aplab.be` tenant is a Microsoft 365 A5 Educational tenant shared by multiple Syntrapxl projects.

The NetBird quickstart script (`getting-started-with-zitadel.sh`) installs Zitadel as the primary OIDC issuer automatically.

## Options considered

| Option | Pro | Con |
|--------|-----|-----|
| **Direct Entra ID in NetBird Dashboard** | Simpler chain; one fewer component | Quickstart script installs Zitadel regardless; bypassing it requires manual NetBird setup |
| **Zitadel as broker with Entra ID external IdP** | Follows quickstart; adds user management layer (roles, groups) independent of Entra ID; central admin point | Two-hop OIDC chain: NetBird → Zitadel → Entra ID |

## Decision

Zitadel as primary OIDC issuer (installed by quickstart script), with Entra ID connected as an external IdP to Zitadel.

This was an architectural reality of the quickstart script, not a deliberate choice between options. The quickstart produces a working stack with Zitadel as issuer. Entra ID is added afterward as an external IdP in Zitadel's console. The resulting chain is architecturally sound: Zitadel provides a central user management layer, and Entra ID provides school account authentication.

**CA policies still work:** Entra ID Conditional Access evaluates at the Entra ID `/authorize` endpoint. The user authenticates directly with Entra ID whether the redirect originates from Zitadel or from the NetBird Dashboard, so CA fires regardless. App-targeted policies do not fire on these OIDC sign-ins (CA matches on the token resource, Microsoft Graph), so the policies target **All resources** and scope by persona group instead.

> **App-registration switch (V30, May 2026).** The earlier build reused a shared app registration (`cebe0d74-be9f-49ac-9f35-65f11586c1bb`), which created a hidden coupling between the sandbox and the team-project stack. V30 created a dedicated `2ITCSC1A-Netbird-Sandbox` registration (`11803ee8-eb15-462c-a286-5415c17a29c6`) and re-pointed Zitadel's Entra ID federation at it. The sandbox-specific registration is the current value; `cebe0d74…` is superseded.

**TLS:** The quickstart attempts Let's Encrypt for `netbird.sandbox.local` — this fails (non-public hostname). Caddy's `tls internal` directive (built-in local CA) is used instead. OIDC requires HTTPS, not a publicly trusted certificate.

## Consequences

- All five CA policies target **All resources** (not a Zitadel or NetBird app registration); blast radius is contained by user-scoping to the persona groups, with `2itcsc1a_admin1` as break-glass exclude
- New users authenticating via Entra ID appear as "pending" in NetBird Dashboard — manual approval required (security feature of the quickstart)
- Zitadel uses nested `roles` JSON; NetBird expects a flat `groups` array — groups claim transformation may be needed in Zitadel
- `docker compose up -d caddy` (not `restart`) is required when volume mounts change
- Because the `aplab.be` tenant is shared, blast radius is contained by **user-scoping** (persona-group include, `2itcsc1a_admin1` break-glass exclude) rather than app-targeting — an app-scoped policy never fires on these OIDC sign-ins because CA matches on the token resource (Microsoft Graph)

## Implementation detail: groups-claim via two Zitadel Actions

The groups-claim mechanism that powers the entire identity chain is implemented via two Zitadel Actions (v1):

- **Action 1 (External Auth):** Receives the Entra ID token at login, maps the Entra display-name string → clean name via allowlist (`2ITCSC1A-Studenten` → `Studenten`), writes to user metadata `sase_groups`
- **Action 2 (Complement Token):** Reads `sase_groups` metadata at token issuance, injects `groups` claim into JWT via `setClaim`

Both actions have `allowed-to-fail: true` — a deliberate decision. Fail-closed enforcement belongs in the policy layer (NetBird ACLs, Squid persona-ACLs), not in the authentication action. An action failure should degrade identity enrichment, not lock out all users.

See: [Decision: GroupSync Pad B](../decisions/groupsync-pad-b.md)

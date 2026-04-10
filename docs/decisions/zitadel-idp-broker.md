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

**CA policies still work:** Entra ID Conditional Access evaluates at the Entra ID `/authorize` endpoint targeted at the NetBird app registration (`cebe0d74-be9f-49ac-9f35-65f11586c1bb`). Whether the redirect originates from Zitadel or from the NetBird Dashboard is irrelevant — the user authenticates directly with Entra ID, and CA fires for that app registration.

**TLS:** The quickstart attempts Let's Encrypt for `netbird.sandbox.local` — this fails (non-public hostname). Caddy's `tls internal` directive (built-in local CA) is used instead. OIDC requires HTTPS, not a publicly trusted certificate.

## Consequences

- All CA-targeted policies must reference the NetBird app registration (`cebe0d74-be9f-49ac-9f35-65f11586c1bb`), not a Zitadel registration
- New users authenticating via Entra ID appear as "pending" in NetBird Dashboard — manual approval required (security feature of the quickstart)
- Zitadel uses nested `roles` JSON; NetBird expects a flat `groups` array — groups claim transformation may be needed in Zitadel
- `docker compose up -d caddy` (not `restart`) is required when volume mounts change
- CA policies must be app-specific (targeting the NetBird app registration only) because the `aplab.be` tenant is shared — tenant-wide policies would affect other projects and students

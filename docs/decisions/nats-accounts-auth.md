---
title: "Decision: NATS accounts{} Authentication for JetStream"
tags: [decision, nats, jetstream, authentication]
---

# Decision: NATS accounts{} Authentication for JetStream

**Status:** Implemented  
**Date:** May 2026 (Verslag32)

## Context

NATS requires an authentication model to secure access to JetStream streams and consumers. The Control Daemon and the NATS→Wazuh forwarder consume from JetStream, requiring `$JS.API.>` and `$JS.ACK.>` access for consumer management and message acknowledgment, while producers (the detection components and the Identity Bridge) only publish to their own subjects plus `_INBOX.>`. Two authentication models exist in NATS: the simpler `authorization{}` block (single-account) and the more capable `accounts{}` block (multi-account with explicit permission grants).

## Options considered

| Option | Pro | Con |
|--------|-----|-----|
| **`authorization{}` (single-account)** | Simpler configuration; fewer lines in `nats-server.conf` | Silently denies JetStream API operations on `$JS.API.>`. No error messages — operations simply fail without explanation. Empirically confirmed as non-functional for JetStream |
| **`accounts{}` (multi-account)** | Full JetStream API access via explicit `$JS.API.>` permission grants; supports account-level isolation | More verbose configuration; users must be defined within named accounts |

## Decision

The `accounts{}` model is used for NATS authentication. This was determined empirically: the `authorization{}` model silently denied all JetStream operations despite seemingly correct permission configuration. Switching to `accounts{}` with explicit `$JS.API.>` publish/subscribe permissions immediately resolved the issue.

All NATS users (Identity Bridge, Control Daemon) are defined within a dedicated account that has explicit permissions for JetStream API subjects, application-specific subjects, and delivery subjects.

## Consequences

- All NATS users must be defined within named accounts in `nats-server.conf`. The top-level `authorization{}` block cannot be used alongside `accounts{}`.
- Secrets (user passwords/tokens) are injected via environment variable interpolation using hex-encoded values, not base64. NATS supports `$ENV_VAR` syntax in its configuration file.
- Adding a new NATS client requires defining the user within the appropriate account and granting subject-level permissions explicitly.
- The `accounts{}` model provides account-level isolation as a side benefit — if future components need separate permission domains, additional accounts can be added without restructuring.

See also: [Component: NATS](../components/nats-jetstream.md), [Component: Identity Bridge](../components/identity-bridge.md), [Component: Control Daemon](../components/control-daemon.md)

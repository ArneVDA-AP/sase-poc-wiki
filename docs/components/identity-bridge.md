---
title: "Identity Bridge"
tags: [identity-bridge, netbird, squid, entra-id, fastapi, zero-trust]
---

# Identity Bridge

**Role:** FastAPI service on mgmt01 that maps NetBird WireGuard overlay IPs to Entra ID persona groups in real-time, enabling Squid to perform identity-based filtering.  
**Version:** Python 3.12 (FastAPI 0.115.0, uvicorn 0.30.0, httpx 0.27.0)  
**Config location:** mgmt01 Docker, `/opt/identity-bridge/`, shared secret in `.env`

## How it works in this stack

The Identity Bridge closes the gap between the ZTNA transport layer (NetBird) and the SWG inspection layer (Squid). Without it, Squid sees only overlay IP addresses — it cannot determine which user or persona group is making a request.

The bridge polls the NetBird Management API (`/api/peers` + `/api/users`) every 30 seconds via a service-user PAT with admin role. It builds an in-memory cache mapping overlay IPs to persona groups. Squid queries this cache via `external_acl_type` for every HTTP request, using the client's overlay IP (`%SRC`) as the lookup key.

The response flow: Squid sends `<IP> <group>` → helper queries `GET /member?ip=<overlay_ip>` → Identity Bridge returns `OK user=<email>` or `ERR` → Squid's persona-ACL fires or falls through.

## Configuration

- **Service user:** `identity-bridge` with admin role — regular user PATs trigger issue #3127 (stripped JWT-propagated auto-groups from all peers at every poll)
- **Cache TTL:** 30s (Squid `external_acl_type ttl=30 negative_ttl=10 children-max=5`)
- **Endpoint:** `GET /member?ip=<overlay_ip>` — returns persona group or ERR
- **Health:** `GET /health`
- **Only connected peers** are cached — disconnected peers fail open to GUI-generated rules
- **Fail-open design:** When unreachable, no identity-based ACLs match, traffic falls through to non-identity URL filtering. All other security layers remain active.

## Integration points

| Interface | Direction | Protocol | Details |
|-----------|-----------|----------|---------|
| NetBird Management API | Outbound (poll) | HTTPS (overlay 100.x.x.x) | `/api/peers` + `/api/users`, PAT auth |
| Squid external_acl | Inbound (per request) | HTTP (management LAN) | `192.168.122.23:8088` |
| NATS JetStream | Outbound (publish) | NATS (sase-internal Docker network) | `identity.login`, `identity.group_change` subjects |

## Known issues / gotchas

- **Overlay IP instability:** NetBird overlay IPs are not stable across re-enrollment. Addendum H hardcoded `.95.98` which became `.218.100` after re-enrollment. Cache invalidation must be peer-ID based, not IP as primary key. See [Finding: Overlay IP instability](../findings/overlay-ip-instability.md).
- **Squid overlay bind-race:** Squid tries to bind the overlay listener on `100.x.x.x` before NetBird wt0 has assigned the overlay IP → errno 49 (address not available). See [Finding: Squid overlay bind-race](../findings/squid-overlay-bind-race.md).
- **Service PAT required:** Regular user PATs trigger NetBird issue #3127. See [Finding: NetBird issue #3127](../findings/netbird-issue-3127.md).

## Related

- [Decision: NetBird Service PAT](../decisions/netbird-service-pat.md)
- [Component: NetBird](netbird.md)
- [Component: Squid](squid.md)
- [Component: NATS JetStream](nats-jetstream.md)
- [Concept: Identity Flow](../concepts/identity-flow.md)

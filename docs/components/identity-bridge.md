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

The response flow: Squid sends `<IP> <group>` → helper queries `GET /lookup?ip=<overlay_ip>` (with the `X-Bridge-Secret` header) → Identity Bridge returns the peer's full group membership → the helper answers `OK user=<email>` if the requested group matches, else `ERR` → Squid's persona-ACL fires or falls through. The bridge itself does not select a persona; persona resolution happens in Squid's `http_access` order (V31).

## Configuration

- **Service user:** `identity-bridge` with admin role — regular user PATs trigger issue #3127 (stripped JWT-propagated auto-groups from all peers at every poll)
- **Cache TTL:** 30s (Squid `external_acl_type ttl=30 negative_ttl=10 concurrency=0`)
- **Endpoint:** `GET /lookup?ip=<overlay_ip>` — requires the `X-Bridge-Secret` header (env `LOOKUP_SECRET`; fail-secure: an empty secret rejects every request). Returns the peer's full group membership (`status`/`user`/`groups`/`os`) or ERR
- **Health:** `GET /health` — open, no auth, leaks no identity
- **Only connected peers** are cached — disconnected peers fail open to GUI-generated rules
- **Fail-open design:** When unreachable, no identity-based ACLs match, traffic falls through to non-identity URL filtering. All other security layers remain active.

## Integration points

| Interface | Direction | Protocol | Details |
|-----------|-----------|----------|---------|
| NetBird Management API | Outbound (poll) | HTTP (NetBird Docker network) | `/api/peers` + `/api/users` via `management:80`, service-PAT auth |
| Squid external_acl | Inbound (per request) | HTTP (management LAN) | `192.168.122.23:8088` |
| NATS JetStream | Outbound (publish) | NATS (sase-internal Docker network) | `identity.peer.connected`, `identity.peer.disconnected`, `identity.multi_persona` subjects |

## Known issues / gotchas

- **Overlay IP instability:** NetBird overlay IPs are not stable across re-enrollment. Addendum H hardcoded `.95.98` which became `.218.100` after re-enrollment. Cache invalidation must be peer-ID based, not IP as primary key. See [Finding: Overlay IP instability](../findings/overlay-ip-instability.md).
- **Squid overlay bind-race:** Squid tries to bind the overlay listener on `100.x.x.x` before NetBird wt0 has assigned the overlay IP → errno 49 (address not available). See [Finding: Squid overlay bind-race](../findings/squid-overlay-bind-race.md).
- **Service PAT required:** Regular user PATs trigger NetBird issue #3127. See [Finding: NetBird issue #3127](../findings/netbird-issue-3127.md).

## Related

- [Decision: NetBird Service PAT](../decisions/netbird-service-pat.md)
- [Component: NetBird](netbird.md)
- [Component: Squid](squid.md)
- [Component: NATS JetStream](nats-jetstream.md)
- [Component: Transparent Proxy](transparent-proxy.md) — parallel-stack capture path that would extend Identity Bridge coverage to non-proxy traffic
- [Concept: Identity Flow](../concepts/identity-flow.md)

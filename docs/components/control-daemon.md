---
title: "Control Daemon"
tags: [control-daemon, nats, netbird, quarantine, threat-scoring]
---

# Control Daemon

**Role:** Python daemon on mgmt01 that consumes the NATS event bus, maintains per-peer threat scores via sliding-window decay, and quarantines peers by removing them from policy-bearing persona groups (deny-by-default).  
**Version:** Python 3.12 (nats-py >= 2.13.0, redis >= 5.0.0, httpx >= 0.27.0)  
**Config location:** `config/mgmt01/control-daemon/` in repo (authoritative — Addendum J §J.6.7 is outdated)

## How it works in this stack

The control daemon is the real-time enforcement engine. It subscribes to `security.alert.>` and `identity.>` on the NATS bus and processes events through three stages:

1. **Enrichment:** Maps overlay IPs to peer identity via an internal `identity.>` cache (populated from Identity Bridge events)
2. **Scoring:** Dispatches on `producer` field (not subject). Each event type has a configurable weight. Scores use sliding-window decay via Redis sorted sets.
3. **Decision:** When a peer's score exceeds the quarantine threshold (default: 80), the daemon removes the peer from all policy-bearing persona groups via NetBird Groups API. Without persona group membership, deny-by-default blocks all connectivity.

**Current state:** `ENFORCE=false` (dry-run) until demo preparation (Session 11).

## Configuration

- **Scoring weights:** `malware=80` (single-event — crosses threshold 80 alone), `dlp_match=30` (accruing). `proxy_block` removed from scoring (ambient OS noise caused false positives — Windows NCSI/telemetry). IDS events are log-only (C2 beacon response belongs with Zeek/RITA, not bespoke correlation). Validated in V35: docent1 triggered EICAR → score 80/80 → quarantined within seconds → restored.
- **Quarantine mechanism:** Strip persona groups from peer → deny-by-default. NOT via a separate deny-group.
- **Unquarantine:** Restores original groups from Redis backup. Fixed: NetBird returns `peers:null` for empty groups (not `[]`), which crashed unquarantine.
- **Redis:** Threat score store + session state on `redis:7-alpine`, port 6379 localhost only.
- **Policy groups:** `NETBIRD_POLICY_GROUPS=Studenten,Docenten,Admins` — only these are stripped during quarantine. Infrastructure groups are structurally untouchable.

## Integration points

| Interface | Direction | Details |
|-----------|-----------|---------|
| NATS JetStream | Inbound (consumer) | Durable + DeliverPolicy.NEW consumer on `security.alert.>` (survives restarts, no replay of historical events); ephemeral + DeliverPolicy.ALL consumer on `identity.>` (rebuilds the in-memory identity map on every start) |
| NetBird Management API | Outbound | Groups API GET/PUT for quarantine/unquarantine |
| Redis | Bidirectional | Threat scores, session state, group backup |

## Known issues / gotchas

- **`peers:null` coercion:** NetBird returns `null` instead of `[]` for groups with no peers. Fixed with `(group.get("peers") or [])` coercion (V35).
- **`proxy_block` false positives:** Ambient Windows NCSI/telemetry traffic generates proxy block events. Removed from scoring weights (V35).
- **IDS correlation not implemented:** Malware branch covers the same real-time quarantine capability with native attribution; C2 beacon response belongs with Zeek/RITA scope. See [Decision: Control Daemon Scope](../decisions/control-daemon-scope.md).

## Related

- [Decision: Control Daemon Scope](../decisions/control-daemon-scope.md)
- [Component: NATS JetStream](nats-jetstream.md)
- [Component: NetBird](netbird.md)

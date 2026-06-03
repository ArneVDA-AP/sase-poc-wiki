---
title: "Runbook: NATS JetStream Event Bus"
tags: [runbook, nats-jetstream, docker, redis, event-bus]
---

# Runbook: NATS JetStream Event Bus

**Node(s):** mgmt01 (Docker — NATS + Redis + Control Daemon), pop01 (producers)
**Prerequisites:** Docker on mgmt01, pop01 SSH access, Python 3 on pop01
**Status:** Operational

---

## Prerequisites checklist

- [ ] Docker and Docker Compose installed on mgmt01
- [ ] SSH access to pop01
- [ ] Python 3 installed on pop01
- [ ] Suricata operational on pop01 ([Runbook 05](05-ids.md))
- [ ] Squid operational on pop01 ([Runbook 03](03-proxy-wpad.md))
- [ ] DNS RPZ operational ([Runbook 06](06-dns-threat-intel.md))
- [ ] Python DLP ICAP operational ([Runbook 04](04-malware-dlp.md))
- [ ] Identity Bridge operational ([Runbook 09](09-identity-bridge.md))

---

## Step 1: Add NATS and Redis to Docker Compose on mgmt01

Add the following services to the Docker Compose stack on mgmt01:

**NATS 2.14.1:**

- Pin the version explicitly — do not use `:latest`
- Configure using the `accounts{}` auth model (not `authorization{}`)
- Expose port 4222 (client connections) on `192.168.122.23`
- Expose port 8222 (monitoring) on `192.168.122.23`
- Mount a persistent Docker volume for JetStream store

> **Gotcha: JetStream store must be on a persistent Docker volume.** Without a named volume, JetStream data is lost on container recreation. See [Finding: NATS store dir](../findings/nats-store-dir.md).

> **Gotcha: Use `accounts{}` auth model, not `authorization{}`.** The `authorization{}` block does not support per-subject publish/subscribe permissions needed for multi-tenant isolation. See [Decision: NATS accounts auth](../decisions/nats-accounts-auth.md).

**Redis 7.x:**

- Used for threat score storage by the Control Daemon
- Expose only on the Docker network (not externally) unless debugging

---

## Step 2: Verify NATS is running

Start the Docker Compose stack and verify NATS is accepting connections:

```bash
docker compose up -d

# Verify from a nats-box container
docker run --rm -it --network <compose-network> natsio/nats-box nats pub test "hello"
# Expected: no error, message published
```

Check the monitoring endpoint:

```bash
curl http://192.168.122.23:8222/varz
# Expected: JSON with server info, JetStream enabled
```

---

## Step 3: Verify connectivity from pop01

From pop01, confirm that NATS is reachable over the management network:

```bash
echo "PING" | nc -w 2 192.168.122.23 4222
# Expected: response from NATS server (PONG or INFO line)
```

If this fails, check firewall rules on mgmt01 and verify that NATS is bound to `192.168.122.23:4222`, not just `127.0.0.1`.

---

## Step 4: Deploy producers on pop01

Deploy Python-based event producers on pop01. Each producer tails a log file and publishes structured events to a NATS subject:

### Producer: Suricata IDS

- Tails `/var/log/suricata/eve.json`
- Publishes to subject: `security.alert.ids`
- Extracts: alert signature, severity, source/dest IP, SID

### Producer: Squid Proxy

- Tails `/var/log/squid/access.log`
- Publishes to subject: `security.alert.proxy`
- Extracts: URL, client IP, HTTP status, ACL decision

### Producer: DNS RPZ

- Unbound Python module integration
- Publishes to subject: `security.alert.dns`
- Extracts: queried domain, RPZ action (NXDOMAIN/redirect), feed source

Verify each producer is publishing:

```bash
# From a nats-box container, subscribe to see events
nats sub "security.alert.>"
# Trigger a test event (e.g., curl to testmyids.com via proxy)
# Expected: event appears on the subscription
```

---

## Step 5: Deploy producers on mgmt01

### Producer: Python DLP

- Integrated into the ICAP server (already running from [Runbook 04](04-malware-dlp.md))
- Publishes to subject: `security.alert.dlp`
- Extracts: matched pattern, file name, client IP, severity

### Producer: Identity Bridge

- Integrated into the Identity Bridge service ([Runbook 09](09-identity-bridge.md))
- Publishes to subjects:
  - `identity.login` — when a new peer connects
  - `identity.group_change` — when a peer's persona group changes

---

## Step 6: Deploy Control Daemon on mgmt01

The Control Daemon is the central event processor that consumes security events and takes automated response actions.

1. Deploy as a Docker container on mgmt01
2. Configure subscriptions: `security.alert.*` (wildcard)
3. Configure Redis connection for threat score storage

**Threat scoring:**

- Redis-backed sliding-window scoring per peer IP
- Each alert type contributes a configurable score weight
- When a peer's score exceeds the quarantine threshold: the Control Daemon removes the peer from its persona group via the NetBird Groups API
- When the score decays below the restore threshold: the peer is re-added to its persona group

See [Component: Control Daemon](../components/control-daemon.md) for the scoring algorithm and threshold configuration.

---

## Step 7: End-to-end verification

Trigger a real alert and trace it through the entire pipeline:

1. **Trigger:** From mobile01, run `curl -x http://100.70.154.79:3128 http://testmyids.com/`
2. **Suricata:** Check that eve.json records the alert (SID 2100498)
3. **Producer:** Verify the event appears on `security.alert.ids` subject
4. **Redis:** Check the threat score for the peer IP has increased:

```bash
docker exec redis redis-cli GET "threat:<peer-ip>"
```

5. **Control Daemon:** Check Control Daemon logs for the received event and scoring decision

If the score exceeds the quarantine threshold, verify that the peer is removed from its persona group on the NetBird Dashboard.

---

## Checklist

- [ ] NATS 2.14.1 running on mgmt01 with `accounts{}` auth model
- [ ] JetStream store on persistent Docker volume
- [ ] Port 4222 accessible from pop01
- [ ] Port 8222 monitoring endpoint responding
- [ ] Redis running on mgmt01
- [ ] Suricata producer publishing to `security.alert.ids`
- [ ] Squid producer publishing to `security.alert.proxy`
- [ ] DNS RPZ producer publishing to `security.alert.dns`
- [ ] DLP producer publishing to `security.alert.dlp`
- [ ] Identity Bridge publishing to `identity.login` / `identity.group_change`
- [ ] Control Daemon consuming `security.alert.*` and scoring in Redis
- [ ] Quarantine action functional (peer removed from group on threshold breach)

---

## Related

- [Component: NATS JetStream](../components/nats-jetstream.md)
- [Component: Control Daemon](../components/control-daemon.md)
- [Decision: NATS accounts auth](../decisions/nats-accounts-auth.md)
- [Decision: Control Daemon scope](../decisions/control-daemon-scope.md)
- [Finding: NATS store dir](../findings/nats-store-dir.md)
- [Runbook 05: IDS](05-ids.md)
- [Runbook 09: Identity Bridge](09-identity-bridge.md)

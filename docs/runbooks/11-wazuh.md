---
title: "Runbook: Wazuh SIEM"
tags: [runbook, wazuh, siem, docker, nats-jetstream]
---

# Runbook: Wazuh SIEM

**Node(s):** mgmt01 (Docker — Wazuh stack + NATS forwarder), pop01 (Wazuh agent)
**Prerequisites:** [Runbook 10: NATS JetStream](10-nats-jetstream.md) completed (NATS operational), Docker on mgmt01
**Status:** Operational

---

## Prerequisites checklist

- [ ] NATS JetStream operational on mgmt01 ([Runbook 10](10-nats-jetstream.md))
- [ ] Docker and Docker Compose installed on mgmt01
- [ ] SSH access to pop01 for agent installation
- [ ] Sufficient disk space on mgmt01 for OpenSearch indices

---

## Step 1: Deploy Wazuh stack on mgmt01

Add the Wazuh stack to Docker Compose on mgmt01. The stack consists of three services:

| Service | Version | Role |
|---------|---------|------|
| Wazuh Manager | v4.14.5 | Alert processing, rule engine, active response |
| Wazuh Indexer | (OpenSearch) | Event storage and search |
| Wazuh Dashboard | (OpenSearch Dashboards) | Web UI for visualization and investigation |

> **Gotcha: Indexer needs host CPU passthrough.** The Wazuh Indexer requires glibc x86-64-v2 CPU features; the default QEMU `kvm64` model crashes it with illegal-instruction errors. Set the GNS3 node CPU model to `host` for mgmt01 and set `vm.max_map_count=262144`. Startup then takes 2-3 minutes to stabilize (transient `port 9200` errors during reinitialization are normal). See [Finding: Wazuh CPU glibc](../findings/wazuh-cpu-glibc.md).

Start the stack:

```bash
docker compose up -d
```

Wait for all three containers to report healthy:

```bash
docker compose ps
# All three services should show "healthy" or "running"
```

---

## Step 2: Deploy NATS-to-Wazuh forwarder

Deploy a Python container on mgmt01 that bridges NATS events into Wazuh:

1. The forwarder is a **durable PULL consumer** (`DeliverPolicy.NEW`, `AckPolicy.EXPLICIT`) on `security.alert.>`
2. For each received event, it writes a line of NDJSON to the shared `wazuh_nats_ingest` volume (`/ingest/security_alerts.json`). The Wazuh manager tails that file via a `<localfile><log_format>json</log_format></localfile>` block — this is the working ingest path; writing to the manager socket and posting to the Wazuh API were both evaluated and rejected.
3. This creates a **dual-write architecture**: security events flow to both the Control Daemon (for automated response) and Wazuh (for logging, correlation, and investigation) independently

> **Note: Dual-write, not serial.** The NATS forwarder and Control Daemon are independent consumers. If Wazuh is down, the Control Daemon continues to process events and vice versa. Neither depends on the other.

> **Gotcha: reconnect-resilience.** A `--force-recreate` of the NATS container changes its IP and breaks any hanging consumer connection. The forwarder must reconnect with `max_reconnect_attempts=-1` and re-resolve DNS (no cached IP).

Verify the forwarder is running and forwarding:

```bash
# Trigger a test event (e.g., browser EICAR download → security.alert.malware)
# Then confirm the NDJSON line landed in the shared ingest file:
docker exec single-node-wazuh.manager-1 tail -n 5 /ingest/security_alerts.json
# And confirm it indexed: Wazuh Dashboard → Discover, index wazuh-alerts-*, query rule.groups:"sase"
```

---

## Step 3: Deploy Wazuh agent on pop01

Install and configure the Wazuh agent on pop01 (agent ID 001):

1. Install the Wazuh agent package on pop01
2. Configure the agent to connect to the Wazuh Manager on mgmt01
3. Register the agent (agent ID: 001)

Configure host-level log collection. The agent feeds the OPNsense **host** telemetry (it does **not** monitor Suricata/Squid/c-icap/Unbound — those reach Wazuh via their own NATS producers and the forwarder, so the agent has `intrusion_detection_events=false` to avoid double-ingest):

| Feed | Source | Purpose |
|------|--------|---------|
| audit | OPNsense audit log | Configuration/admin actions |
| configd | OPNsense configd | Service/config events |
| filter | OPNsense firewall (filterlog) | Packet-filter events |
| kernel | OPNsense kernel log | System/kernel events |
| pkg | OPNsense package manager | Package install/update events |

4. The agent also has `active_response=false` — Active Response (enforcement) is owned by the Control Daemon (Layer 3), not by the agent on pop01.

Restart the agent and verify registration:

```bash
# On Wazuh Manager (mgmt01)
docker exec single-node-wazuh.manager-1 /var/ossec/bin/agent_control -l
# Expected: agent 001 (pop01) — Active
```

---

## Step 4: Configure custom Wazuh rules

Create custom rules for the bus producers. All hang off a single base rule (100500) that matches on the `producer` field; the children classify and assign severity:

| Rule | Producer / condition | Level |
|------|----------------------|-------|
| 100500 | base — `producer` ∈ {suricata, squid, dlp, c-icap, unbound} | 0 |
| 100501 | Squid proxy event | 5 |
| 100510 | Suricata IDS alert | 6 |
| 100511 | Suricata IDS, severity 1 | 10 |
| 100513 | Suricata IDS, severity 3 | 3 |
| 100520 | DLP match | 10 |
| 100530 | malware (ClamAV / c-icap) | 12 |
| 100540 | DNS RPZ block | 8 |

> **Gotcha:** the children attach to the base via `if_sid 100500`, so the base pcre2 alternation must include every producer. `unbound` was initially missing, so rule 100540 never fired until it was added (V38.9).

Place custom rules in `config/mgmt01/wazuh/local_rules.xml` (mounted into the manager). After editing rules, restart the manager — never `docker restart`:

```bash
docker exec single-node-wazuh.manager-1 /var/ossec/bin/wazuh-control restart
```

---

## Step 5: Configure CASB Layer 2 — M365 API integration

CASB Layer 2 polls Microsoft 365 audit activity and remediates risky shares. There is **no** "Wazuh office365 module" in this setup — a custom producer does the polling:

1. Deploy the `o365_producer` container on mgmt01:
   - Polls the **Office 365 Management Activity API** (`contentType=Audit.SharePoint`) with app-registration credentials for the aplab.be tenant
   - Normalizes each audit blob and publishes it to NATS subject `security.alert.casb` (NATS role `casb-pub`)
   - From there the NATS→Wazuh forwarder carries it into the manager like any other bus event

2. Create custom rules in the 100600-family (a self-contained base, separate from the bus producers' 100500 base):

| Rule | Detection (`producer` = `o365`) | Level |
|------|----------------------------------|-------|
| 100600 | base — `producer` = `o365` | 0 |
| 100601 | `Operation` = AnonymousLinkCreated | 10 |
| 100602 | `Operation` = SharingLinkCreated AND `SharingLinkScope` = Anyone | 10 |
| 100603 | `Operation` = SharingSet AND `TargetUserOrGroupType` = Guest | 8 |

3. Deploy the Active Response scripts (ported from the team's CASB project), behind an `ENFORCE` gate that **defaults to detect-only** (`ENFORCE=false`):
   - `sharepoint_remediate.sh` — triggered by rules **100601, 100602**; revokes the anonymous / anyone link via the Microsoft Graph API
   - `guest_remediate.sh` — triggered by rule **100603**; removes the guest grant
   - Both read Graph credentials from `graph.env` (in the `wazuh_etc` volume, never committed) and log the remediation action

> **Gotcha: jq dependency.** The Active Response scripts need `jq`, which is not in the stock manager image — without it the script fails silently on its first call. Install it persistently via the `install_deps.sh` entrypoint wrapper plus a compose `entrypoint:` override (V39); a plain `yum install` does not survive a recreate.

> **Note:** This is CASB Layer 2 in the three-layer CASB architecture. See [Decision: CASB three layers](../decisions/casb-three-layers.md) for the full design.

---

## Step 6: Verify events in Wazuh Dashboard

1. Open the Wazuh Dashboard in a browser
2. Navigate to **Discover**
3. Search for SASE events:

```
rule.groups: "sase" OR data.alert.signature_id: *
```

**Expected:** Events from Suricata, Squid, DNS RPZ, and DLP appear in the dashboard with properly parsed fields.

> **Gotcha: Dashboard app modules are gated in the air-gapped sandbox.** Discover works fully (it reads the indexer directly), but the dashboard's app modules (Security Events, Agents) show "Status: Offline". The cause is not DNS: the manager's internet-dependent `version/check` returns HTTP 500 in the air-gapped sandbox, which the dashboard's `check-api` cascades into an offline determination even though the manager API itself is healthy (authenticate/info/stats all return 200). Investigate via Discover, not the app modules. See [Finding: Wazuh Dashboard airgate](../findings/wazuh-dashboard-airgate.md).

---

## Step 7: Verify CASB Active Response

**Active Response defaults to detect-only (`ENFORCE=false`).** In detect-only mode the rule fires and the remediation script logs what it *would* do, but no Microsoft Graph API call is made. Set `ENFORCE=true` (the gate inside the script) only when you intend live remediation.

Test the SharePoint remediation flow:

1. Create an anonymous sharing link on a SharePoint document in the aplab.be tenant
2. Wait for the `o365_producer` to poll the Management Activity API and publish the audit event to `security.alert.casb`
3. Wazuh should trigger rule 100601 (`AnonymousLinkCreated`)
4. Active Response script `sharepoint_remediate.sh` should execute (log-only unless `ENFORCE=true`)
5. With `ENFORCE=true`, verify the anonymous sharing link is revoked via the Microsoft Graph API

> **Status: the revoke chain is proven to HTTP-204 via an offline stub; live revocation against a real OneDrive/SharePoint file is pending test-account provisioning** (an external Microsoft dependency). Detect-only mode and the rule→Active-Response trigger are validated; live end-to-end revoke is not yet demonstrated on the sandbox.

Check the Active Response log:

```bash
docker exec single-node-wazuh.manager-1 cat /var/ossec/logs/active-responses.log | tail -20
```

---

## Checklist

- [ ] Wazuh Manager v4.14.5 running on mgmt01
- [ ] Wazuh Indexer (OpenSearch) running and healthy
- [ ] Wazuh Dashboard accessible via browser (Discover; app modules gated in air-gap)
- [ ] NATS-to-Wazuh forwarder consuming `security.alert.>` → NDJSON to `wazuh_nats_ingest`
- [ ] Wazuh agent 001 (pop01) registered and active
- [ ] Custom bus rules 100500–100540 deployed (`local_rules.xml`)
- [ ] Bus rules classify proxy/IDS/DLP/malware/DNS-RPZ events (children attach via `if_sid 100500`)
- [ ] `o365_producer` polling Office 365 Management Activity API → `security.alert.casb`
- [ ] Custom rules 100600-family detecting SharePoint violations
- [ ] Active Response `sharepoint_remediate.sh` (ENFORCE detect-only default; live revoke pending)
- [ ] Events visible in Wazuh Dashboard → Discover

---

## Related

- [Component: Wazuh](../components/wazuh.md)
- [Component: NATS JetStream](../components/nats-jetstream.md)
- [Component: Control Daemon](../components/control-daemon.md)
- [Decision: CASB three layers](../decisions/casb-three-layers.md)
- [Finding: Wazuh CPU glibc](../findings/wazuh-cpu-glibc.md)
- [Finding: Wazuh Dashboard airgate](../findings/wazuh-dashboard-airgate.md)
- [Runbook 10: NATS JetStream](10-nats-jetstream.md)

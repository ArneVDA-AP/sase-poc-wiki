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

> **Gotcha: CPU spike on startup.** The Wazuh Manager and Indexer cause significant CPU load during initial startup due to glibc compatibility issues in the Docker environment. This is a known issue — wait 2-3 minutes for the stack to stabilize. See [Finding: Wazuh CPU glibc](../findings/wazuh-cpu-glibc.md).

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

Deploy a Python script on mgmt01 that bridges NATS events into Wazuh:

1. The forwarder subscribes to `security.alert.*` subjects on NATS
2. For each received event, it forwards to the Wazuh API `/events` endpoint
3. This creates a **dual-write architecture**: security events flow to both the Control Daemon (for automated response) and Wazuh (for logging, correlation, and investigation) independently

> **Note: Dual-write, not serial.** The NATS forwarder and Control Daemon are independent subscribers. If Wazuh is down, the Control Daemon continues to process events and vice versa. Neither depends on the other.

Verify the forwarder is running and forwarding:

```bash
# Trigger a test event (e.g., Suricata alert)
# Then check Wazuh API for the event
curl -k -u <wazuh-api-user>:<password> https://localhost:55000/security/events?limit=5
```

---

## Step 3: Deploy Wazuh agent on pop01

Install and configure the Wazuh agent on pop01 (agent ID 001):

1. Install the Wazuh agent package on pop01
2. Configure the agent to connect to the Wazuh Manager on mgmt01
3. Register the agent (agent ID: 001)

Configure direct log collection — the agent monitors these files:

| Log file | Source | Purpose |
|----------|--------|---------|
| `/var/log/suricata/eve.json` | Suricata IDS | IDS alerts and flow data |
| `/var/log/squid/access.log` | Squid proxy | Proxy access events |

4. Create custom decoders for the SASE event format so Wazuh can parse the structured fields from Suricata and Squid logs

Restart the agent and verify registration:

```bash
# On Wazuh Manager (mgmt01)
docker exec wazuh-manager /var/ossec/bin/agent_control -l
# Expected: agent 001 (pop01) — Active
```

---

## Step 4: Configure custom Wazuh rules

Create custom rules in the 100500-100600 range for SASE-specific detections:

| Rule range | Category | Examples |
|------------|----------|----------|
| 100500-100519 | IDS correlation | High-severity Suricata alerts, repeated alerts from same source |
| 100520-100539 | DLP alerts | Sensitive data pattern matches, blocked uploads |
| 100540-100559 | DNS RPZ blocks | RPZ-blocked domains, repeated block attempts |
| 100560-100599 | Reserved | Future use |

Place custom rules in the Wazuh Manager's local rules directory. After adding rules, restart the Wazuh Manager:

```bash
docker exec wazuh-manager /var/ossec/bin/wazuh-control restart
```

---

## Step 5: Configure CASB Layer 2 — M365 API integration

Set up Wazuh's ms-graph module to poll the Microsoft 365 Management Activity API for cloud security events:

1. Configure the `ms-graph` module in the Wazuh Manager configuration:
   - API endpoint: Microsoft 365 Management Activity API
   - Authentication: App registration credentials for aplab.be tenant
   - Polling interval: configured per API requirements

2. Create custom rules in the 100600-family for M365 violations:

| Rule | Detection | Response |
|------|-----------|----------|
| 100600 | SharePoint anonymous sharing link created | Alert + Active Response |
| 100601 | OneDrive external share detected | Alert |
| 100602 | Suspicious sign-in from unusual location | Alert |

3. Deploy Active Response script `sharepoint_remediate.sh`:
   - Triggered by rule 100600
   - Revokes anonymous/external sharing links via the Microsoft Graph API
   - Logs remediation action back to Wazuh

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

> **Gotcha: Dashboard may fail to load initially.** If the Wazuh Dashboard shows a connection error related to `airgate` hostname resolution, this is a known DNS issue in the Docker environment. See [Finding: Wazuh Dashboard airgate](../findings/wazuh-dashboard-airgate.md).

---

## Step 7: Verify CASB Active Response

Test the SharePoint remediation flow:

1. Create an anonymous sharing link on a SharePoint document in the aplab.be tenant
2. Wait for the ms-graph module to poll and detect the violation
3. Wazuh should trigger rule 100600
4. Active Response script `sharepoint_remediate.sh` should execute
5. Verify: the anonymous sharing link is revoked (no longer accessible)

Check the Active Response log:

```bash
docker exec wazuh-manager cat /var/ossec/logs/active-responses.log | tail -20
```

---

## Checklist

- [ ] Wazuh Manager v4.14.5 running on mgmt01
- [ ] Wazuh Indexer (OpenSearch) running and healthy
- [ ] Wazuh Dashboard accessible via browser
- [ ] NATS-to-Wazuh forwarder subscribing to `security.alert.*`
- [ ] Wazuh agent 001 (pop01) registered and active
- [ ] Custom decoders parsing SASE event format
- [ ] Custom rules 100500-100600 deployed
- [ ] M365 ms-graph module polling Management Activity API
- [ ] Custom rules 100600-family detecting SharePoint violations
- [ ] Active Response `sharepoint_remediate.sh` revoking sharing links
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

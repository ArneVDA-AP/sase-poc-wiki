---
title: "Runbook: ZTNA Overlay"
tags: [runbook, netbird, zitadel, entra-id, wireguard, ztna]
---

# Runbook: ZTNA Overlay

**Source:** `raw/Doc6_NetBird_ZTNA.md`
**Node(s):** mgmt01 (Docker stack) + pop01 (NetBird agent) + all peers
**Prerequisites:** [Runbook 01: Lab Environment](01-lab-environment.md) completed, Entra ID tenant with A5 license
**Status:** Operational — snapshot `Fase2-ZTNA-Complete`

---

## Prerequisites checklist

- [ ] Lab environment fully operational (Runbook 01)
- [ ] Entra ID tenant (`aplab.be`) accessible
- [ ] A5 Educational license active
- [ ] App registration created in Entra ID:
  - Client ID: `cebe0d74-be9f-49ac-9f35-65f11586c1bb`
  - Tenant ID: `23e9bcdc-5cb9-4867-9310-76cc0b462ddc`
- [ ] mgmt01 has Docker installed and running

---

## Step 1: Entra ID App Registration

Create the app registration in Entra ID before deploying NetBird:

```
Entra ID → App Registrations → New Registration
  Name: NetBird SASE PoC
  Supported account types: Accounts in this organizational directory only
```

Note the Client ID and Tenant ID. The redirect URI will be added later (after NetBird deployment, Step 8).

---

## Step 2: Hosts entry on mgmt01

```bash
ssh mgmt@10.158.10.67 -p 7023
```

```bash
echo "192.168.122.23  netbird.sandbox.local" | sudo tee -a /etc/hosts
ping -c 2 netbird.sandbox.local
# Must resolve to 192.168.122.23
```

---

## Step 3: Run quickstart script

```bash
curl -sSLO https://github.com/netbirdio/netbird/releases/latest/download/getting-started-with-zitadel.sh
chmod +x getting-started-with-zitadel.sh
sudo ./getting-started-with-zitadel.sh
```

When prompted:
```
Enter the domain you want to use for NetBird: netbird.sandbox.local
Select your Identity Provider setup: [0] None (use Zitadel built-in)
```

> **Gotcha: The script will fail — this is expected.** It tries to contact Let's Encrypt for `netbird.sandbox.local` and fails the ACME challenge (private hostname). Stop with **Ctrl+C** and continue to Step 4.

---

## Step 4: Fix Caddyfile for `tls internal`

```bash
nano Caddyfile
```

Add `tls internal` to every server block referencing `netbird.sandbox.local`:

```caddyfile
netbird.sandbox.local {
    tls internal

    # ... existing reverse_proxy configuration
}

netbird.sandbox.local:443 {
    tls internal

    # ... existing configuration
}
```

See [Decision: Zitadel as IdP broker](../decisions/zitadel-idp-broker.md) for why self-signed TLS is acceptable here.

---

## Step 5: Add Docker network alias

```bash
nano docker-compose.yml
```

Find the `caddy` service and add a network alias:

```yaml
services:
  caddy:
    networks:
      netbird:
        aliases:
          - netbird.sandbox.local
```

This lets internal containers (Zitadel, management) resolve the hostname via this alias.

---

## Step 6: Restart stack and validate

```bash
sudo docker compose down
sudo docker compose up -d
```

Wait 60 seconds for all services to initialize:

```bash
sudo docker compose ps
# All containers must show status "Up"

curl -k https://netbird.sandbox.local/zitadel/debug/ready
# Must return "ok" or HTTP 200
```

Verify `management.json` — all URL references must use `https://netbird.sandbox.local`, not bare IPs:

```bash
cat management.json | grep -i "netbird.sandbox.local"
```

> **Gotcha: Docker volume mounts require container recreation, not restart.** `docker compose restart caddy` does NOT apply new volume mounts. Always use `docker compose up -d caddy`.
> See [Finding: Docker volume recreation](../findings/docker-volume-recreation.md).

---

## Step 7: Hosts entries on all NetBird clients

Every machine that communicates with NetBird must resolve the hostname:

| Machine | Method |
|---------|--------|
| pop01 (OPNsense) | Console → option 8 (shell): `echo "192.168.122.23 netbird.sandbox.local" >> /etc/hosts` |
| mobile01 (Windows) | As Administrator: edit `C:\Windows\System32\drivers\etc\hosts`, add `192.168.122.23 netbird.sandbox.local` |
| site01 (VyOS) | `set system static-host-mapping host-name netbird.sandbox.local inet 192.168.122.23` |

On the GNS3 host: `netbird.sandbox.local` is routed via the nginx SNI stream passthrough → `192.168.122.23:443`.

**Verify:** From each machine, `ping netbird.sandbox.local` or equivalent resolves correctly.

---

## Step 8: Link Entra ID as external IdP in Zitadel

Navigate to the Zitadel console:

```
https://netbird.sandbox.local/zitadel/ui/console
→ Settings → Identity Providers → Add → Microsoft Azure AD
→ Client ID: cebe0d74-be9f-49ac-9f35-65f11586c1bb
→ Client Secret: <from Entra ID app registration>
→ Tenant ID: 23e9bcdc-5cb9-4867-9310-76cc0b462ddc
```

In Entra ID → App registration → Authentication: add the redirect URI that Zitadel generates.

**User Approval Flow:** New users logging in via Entra ID appear as "pending" in NetBird Dashboard → Users. An admin must manually approve them. This is a deliberate security feature, not a bug.

**Groups sync:** Zitadel uses `roles` (nested JSON) while NetBird expects a flat `groups` array. If group memberships don't appear, check the Zitadel Action script for groups-claim transformation.

---

## Step 9: Install NetBird agent on pop01

From the pop01 console (FreeBSD), install the NetBird client. The WireGuard interface `wt0` is created automatically.

> **Gotcha: `config.json` becomes 0 bytes after unclean shutdown.** FreeBSD's UFS with soft updates can leave an empty file when QEMU is killed abruptly. NetBird won't start, `wt0` doesn't exist, and all overlay listeners fail.
> See [Finding: NetBird config zero bytes](../findings/netbird-config-zero-bytes.md).

**Mitigation — back up after every session:**

```bash
cp /var/db/netbird/config.json /var/db/netbird/config.json.bak
```

Recovery: if `config.json` is 0 bytes, restore from backup and restart NetBird.

---

## Step 10: Create groups in NetBird Dashboard

NetBird Dashboard → Peers → create groups:

| Group | Members | Role |
|-------|---------|------|
| `SASE-Admins` | pop01, mgmt01 | Administrative SSH/HTTPS access |
| `SASE-MobileUsers` | mobile01, future BYOD | BYOD clients |
| `SASE-Services` | pop01, mgmt01 | Service endpoints (WPAD, DNS, proxy) |
| `SASE-InternalResources` | (resource group for Network) | DC-LAN resources |

mgmt01 is in both `SASE-Admins` and `SASE-Services` — this separates the infra role (admin access) from the service role (WPAD via Caddy, ioc2rpz feeds).

---

## Step 11: Create ACL policies

> **Gotcha: Order is critical.** Create all policies **before** deleting the default all-to-all policy. If you delete the default without replacements, all peer communication drops immediately.

**Policy 1 — Admin-Infrastructure:**

```
Name:         Admin-Infrastructure
Sources:      SASE-Admins
Destinations: SASE-Admins
Protocol:     All
Action:       Accept
```

**Policy 2 — Mobile-to-Services:**

```
Name:         Mobile-to-Services
Sources:      SASE-MobileUsers
Destinations: SASE-Services
Protocol:     All
Action:       Accept
```

**Policy 3 — Datacenter Access:**

```
Name:         Datacenter Access
Sources:      SASE-MobileUsers
Destinations: SASE-InternalResources
Protocol:     All
Action:       Accept
```

---

## Step 12: Configure exit node via Network Routes

NetBird Dashboard → Network Routes → Add Route:

```
Network:      0.0.0.0/0
Routing Peer: pop01
Description:  Internet exit node
Groups:       SASE-MobileUsers
```

This routes all non-overlay traffic from BYOD clients through pop01 — required for Squid to inspect the traffic.

---

## Step 13: Configure DC-LAN via Networks

NetBird Dashboard → Networks → Create Network:

```
Name: Internal-DC
```

Add routing peer: pop01. Add resource:

```
Name:  DC-LAN
Type:  IP Range
Range: 10.0.0.0/8
Group: SASE-InternalResources
```

See [Decision: CA + Posture hybrid](../decisions/ca-posture-hybrid.md) for why Networks (ACL-aware) is used for DC-LAN instead of Network Routes.

**Verify:** Network Routes table shows `Internal-DC` as active with pop01 online (green indicator).

---

## Step 14: Configure NetBird DNS

NetBird Dashboard → DNS:

**Custom DNS zone:**
```
Domain:     sandbox.local
Nameserver: pop01 (100.70.154.79)
```

**Primary nameserver:**
```
Primary nameserver: pop01 (100.70.154.79)
Match domains:      (LEAVE EMPTY)
```

> **Gotcha: Empty match-domains is critical.** Empty means pop01 becomes the primary nameserver for **all** DNS queries — not just `*.sandbox.local`. Without this, external queries bypass Unbound and DNS RPZ protection doesn't cover external domains.
> See [Finding: NetBird primary nameserver](../findings/netbird-primary-nameserver.md).

---

## Step 15: Validate on mobile01

```powershell
# NetBird status
netbird status
# Expected: Connected, X/Y peers, routes active

# Datacenter reachable
Test-NetConnection 10.0.0.100 -Port 80
# Expected: TcpTestSucceeded: True

# Internet via exit node
Test-NetConnection 8.8.8.8
# Expected: PingSucceeded: True
```

---

## Final verification

- [ ] `curl -k https://netbird.sandbox.local/zitadel/debug/ready` returns HTTP 200
- [ ] Browser to `https://netbird.sandbox.local` shows Zitadel login page
- [ ] Login with Entra ID test account works (MFA prompt appears)
- [ ] NetBird Dashboard accessible, peers visible (green = online)
- [ ] `netbird status` on mobile01 shows "Connected"
- [ ] Ping between overlay IPs: `100.70.154.79`, `100.70.135.241`, `100.70.95.98`
- [ ] DC-LAN reachable from mobile01 (`10.0.0.100`)
- [ ] Internet reachable from mobile01 via exit node
- [ ] `config.json` backup created on pop01

---

## Related

- [Component: NetBird](../components/netbird.md)
- [Component: Caddy](../components/caddy.md)
- [Concept: Zero Trust](../concepts/zero-trust.md)
- [Decision: Zitadel as IdP broker](../decisions/zitadel-idp-broker.md)
- [Finding: NetBird config zero bytes](../findings/netbird-config-zero-bytes.md)
- [Finding: Docker volume recreation](../findings/docker-volume-recreation.md)
- [Finding: NetBird primary nameserver](../findings/netbird-primary-nameserver.md)

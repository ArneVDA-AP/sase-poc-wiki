---
title: "VyOS — SASE Gateway (site01)"
tags: [network, sase, sd-wan, nat, zero-trust]
---

# VyOS — SASE Gateway (site01)

**Role:** SASE Gateway for the remote site — provides WAN connectivity and NAT for the Site-LAN, filling the SD-WAN pillar via a Zero Trust Branch model. Equivalent to a Zscaler Zero Trust Branch edge appliance.  
**Version:** VyOS rolling 2026.02.16  
**Status:** ✅ Operational — site01 at `192.168.122.33`, sitepc01 (Tiny11) enrolled in sandbox  
**Config location:** `/etc/vyos/config.boot` on site01

---

## How it works in this stack

VyOS runs as `site01` with two interfaces: eth0 on the WAN segment (`192.168.122.33`) and eth1 on the Site-LAN (`172.16.10.1/24`). Its responsibilities are deliberately limited to connectivity — it acts as a NAT gateway so that devices on the Site-LAN can reach the internet, but it enforces no security policy itself.

The branch follows a **Zero Trust Branch model**: the entire site is treated as an untrusted location, no different from a coffee-shop Wi-Fi network. There are no IPsec tunnels between site01 and pop01. Instead, sitepc01 runs a [NetBird](netbird.md) client and joins the WireGuard overlay with an identity-based connection, following exactly the same path as any mobile user. All security inspection happens at the PoP, not at the branch edge.

This mirrors Zscaler's Zero Trust SD-WAN architecture, where the branch appliance provides raw connectivity while the Zero Trust Exchange (here: pop01's SWG chain) provides inspection and policy enforcement.

**IP addressing:**

| Interface | Network | Address |
|-----------|---------|---------|
| eth0 (WAN) | Switch-WAN `192.168.122.0/24` | `192.168.122.33` (gw `192.168.122.1`) |
| eth1 (Site-LAN) | Switch-Site `172.16.10.0/24` | `172.16.10.1` |

External SSH access is available via a GNS3 host port-forward at `10.158.10.67:7033`.

---

## Configuration

### NAT masquerade

Site-LAN traffic is NATed through eth0 so that branch devices reach the WAN:

```
set nat source rule 10 outbound-interface name eth0
set nat source rule 10 source address 172.16.10.0/24
set nat source rule 10 translation address masquerade
```

### Static host mapping for NetBird

VyOS provides a local DNS override so that branch devices can resolve the NetBird coordination server before the overlay is up:

```
set system static-host-mapping host-name netbird.sandbox.local inet 192.168.122.23
```

### QoS shaper (DSCP-based)

VyOS applies a QoS shaper on eth0 egress that classifies Microsoft Teams media by DSCP marking — the SD-WAN QoS layer of the project. The DSCP values are set at the endpoint (Intune NetworkQoSPolicy CSP / GPO); VyOS sees them in cleartext on the **split-tunnel path**, where Teams Optimize media is routed to eth1 directly (via an Intune `/14` route) rather than through the WireGuard overlay. All other traffic stays inside the encrypted overlay, where app-aware QoS is not possible.

On VyOS rolling 2026.02.16 the shaper uses the `qos policy shaper` command tree (not the older `traffic-policy`):

```
set qos policy shaper SDWAN-WAN bandwidth '20mbit'
set qos policy shaper SDWAN-WAN default bandwidth '50%'
set qos policy shaper SDWAN-WAN default queue-type 'fq-codel'

# class 10 — Teams audio (DSCP EF, strict priority)
set qos policy shaper SDWAN-WAN class 10 bandwidth '10%'
set qos policy shaper SDWAN-WAN class 10 ceiling '30%'
set qos policy shaper SDWAN-WAN class 10 priority '0'
set qos policy shaper SDWAN-WAN class 10 match TEAMS-AUDIO ip dscp 'EF'

# class 15 — Teams video (DSCP AF41)
set qos policy shaper SDWAN-WAN class 15 priority '1'
set qos policy shaper SDWAN-WAN class 15 match TEAMS-VIDEO ip dscp 'AF41'

# class 18 — Teams screen-sharing (DSCP AF21)
set qos policy shaper SDWAN-WAN class 18 priority '2'
set qos policy shaper SDWAN-WAN class 18 match TEAMS-SHARING ip dscp 'AF21'

set qos interface eth0 egress 'SDWAN-WAN'
```

The 20 Mbit root bandwidth is a deliberate demo-ceiling: the libvirt path has no real WAN bottleneck, so a low ceiling is needed to fabricate contention for the under-load test. Production aligns this to the real WAN capacity.

**Verification:** the `show qos interface eth0` operational command does not exist on this rolling build — use the underlying Linux `tc` instead:

```
sudo tc -s class show dev eth0
```

**Proven (V43):** a synthetic DSCP-EF stream marked 300/300 packets into class 10 with 0 drops (contract B4). Under load (Test #5), the EF class held 0 drops / 0 overlimits while the bulk default class took 26 drops + ~17,000 overlimits — real-time media is protected under congestion. See [Decision: ZT SD-WAN Branch](../decisions/zt-sdwan-branch.md).

> **Build note:** on this rolling build a percentage `ceiling` maps incorrectly into `tc` (`ceiling '30%'` becomes `ceil 3Gbit`, effectively unbounded). Classification and priority work correctly; only ceiling enforcement is affected — benign for the PoC demo.

### WAN link monitoring

A health-check script (`/config/scripts/wan-health-check.sh`) runs on a 30-second VyOS task-scheduler interval, pinging pop01 (`192.168.122.13`), the internet gateway (`192.168.122.1`), and `8.8.8.8`. On failure it logs a `CRITICAL` line to `/var/log/sdwan-health.log`.

**Proven (V43, Test #6):** taking pop01's interface down produced a `CRITICAL` detection within 30 seconds, with recovery logged once the interface returned. This is honest single-WAN framing — detection and alerting, not an automatic dual-WAN switch (the lab has a single WAN). Production would add multi-PoP failover, BFD/SNMP, and fail-closed enforcement.

---

## Integration points

- **sitepc01 traffic path:** sitepc01 → NAT via site01 → NetBird overlay → pop01 (SWG inspection) for Default/Allow traffic. This is the same path mobile01 follows — no branch-specific tunnel exists. Teams Optimize media instead takes the split-tunnel path direct out of eth0 (see QoS above).
- **Site-LAN segment:** `172.16.10.0/24`. sitepc01 sits at `172.16.10.10` with site01 as its default gateway. It also carries a temporary second NIC on the WAN segment (`192.168.122.50`), used as a setup/RDP scaffold — to be removed once overlay-RDP is proven.
- **No direct WireGuard tunnel** between site01 and pop01. All security enforcement is centralised at the PoP.
- **sitepc01 identity (two independent layers):** enrolled in NetBird as `docent1` (Docenten persona); separately Entra ID joined and Intune enrolled via `student1`, the only Intune-licensed sandbox account (`docent1`/`admin1` are deliberately unlicensed). Documented in V44.
- **NetBird coordination:** sitepc01 uses the [NetBird](netbird.md) client to join the overlay, authenticated through Entra ID.

---

## Known issues / gotchas

### DNS bootstrap for sitepc01 enrollment

sitepc01 required a temporary scaffold-NIC (Ethernet 2 pointed at `8.8.8.8`) during initial enrollment because [Unbound](ioc2rpz.md) on pop01 blocks DNS queries from non-overlay IP addresses via its WAN ACL. Without the scaffold-NIC, the machine cannot resolve Entra ID or NetBird endpoints to complete enrollment.

### ICMP misleading under ALLOW-only policy

Ping to overlay peers fails by design when the firewall policy model is ALLOW-only (no ICMP rule). Use port-based connectivity tests instead:

```powershell
Test-NetConnection -ComputerName <target> -Port 3128
```

### Caddy CA certificate chicken-and-egg

The Caddy internal CA certificate must be imported on sitepc01 before NetBird enrollment can succeed, but the certificate cannot be transferred over the overlay because the overlay is not yet up. The workaround is to transfer the certificate via the scaffold-NIC before the overlay is established.

### Classic IPsec SD-WAN descoped (QoS and failover are implemented)

Only the *classic* IPsec/uCPE SD-WAN approach was descoped — the original acceptance tests F12–F14 (IPsec tunnel, site-to-site datacenter access) are marked N/A. The SD-WAN QoS and failover-detection capabilities are **implemented** under the Zero Trust Branch model (the DSCP shaper and WAN health-check above, proven in V43 Test #5 and Test #6). The rationale is documented in [Decision: SD-WAN Descoped](../decisions/sdwan-descoped.md) and [Decision: ZT SD-WAN Branch](../decisions/zt-sdwan-branch.md).

---

## Related

- [Architecture overview](../overview/architecture.md)
- [Component: NetBird](netbird.md)
- [Component: GNS3](gns3.md)
- [Concept: SASE](../concepts/sase.md)
- [Decision: SD-WAN Descoped](../decisions/sdwan-descoped.md)
- [Decision: ZT SD-WAN Branch](../decisions/zt-sdwan-branch.md)
- [Finding: DC-LAN isolation route ACL](../findings/dc-lan-isolation-route-acl.md)

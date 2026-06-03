---
title: "VyOS — SASE Gateway (site01)"
tags: [network, sase, sd-wan, nat, zero-trust]
---

# VyOS — SASE Gateway (site01)

**Role:** SASE Gateway for the remote site — provides WAN connectivity and NAT for the Site-LAN, filling the SD-WAN pillar via a Zero Trust Branch model. Equivalent to a Zscaler Zero Trust Branch edge appliance.  
**Version:** VyOS 1.4 (rolling)  
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

### DSCP / QoS (planned)

DSCP EF marking for Microsoft Teams traffic was planned to demonstrate the QoS layer of SD-WAN but has not yet been implemented. Classic IPsec and advanced QoS SD-WAN features were descoped in favour of the Zero Trust Branch approach; see [Decision: SD-WAN Descoped](../decisions/sdwan-descoped.md).

---

## Integration points

- **sitepc01 traffic path:** sitepc01 → NAT via site01 → NetBird overlay → pop01 (SWG inspection). This is the same path mobile01 follows — no branch-specific tunnel exists.
- **Site-LAN segment:** `172.16.10.0/24`. sitepc01 sits at `172.16.10.50` with site01 as its default gateway.
- **No direct WireGuard tunnel** between site01 and pop01. All security enforcement is centralised at the PoP.
- **sitepc01 identity:** enrolled as `docent1` (Docenten persona) — Entra ID joined and Intune enrolled (documented in V44).
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

### SD-WAN features descoped

Classic SD-WAN capabilities such as IPsec tunnelling and QoS marking were descoped from the project. The rationale is documented in [Decision: SD-WAN Descoped](../decisions/sdwan-descoped.md) and replaced by the Zero Trust Branch model described in [Decision: ZT SD-WAN Branch](../decisions/zt-sdwan-branch.md).

---

## Related

- [Architecture overview](../overview/architecture.md)
- [Component: NetBird](netbird.md)
- [Component: GNS3](gns3.md)
- [Concept: SASE](../concepts/sase.md)
- [Decision: SD-WAN Descoped](../decisions/sdwan-descoped.md)
- [Decision: ZT SD-WAN Branch](../decisions/zt-sdwan-branch.md)
- [Finding: DC-LAN isolation route ACL](../findings/dc-lan-isolation-route-acl.md)

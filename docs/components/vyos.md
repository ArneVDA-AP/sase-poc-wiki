---
title: "VyOS — SD-WAN Gateway (site01)"
tags: [network, sase, firewall]
---

# VyOS — SD-WAN Gateway (site01)

**Role:** SASE Gateway for the remote site — provides WAN connectivity and NAT for the Site-LAN segment, filling the SD-WAN pijler of the SASE stack. Equivalent to a Zscaler Zero Trust Branch edge appliance.  
**Status:** ✅ Deployed — site01 operational at `192.168.122.33`. sitepc01 (client node on Site-LAN) has no OS installed.  
**Config location:** `/etc/vyos/config.boot` on site01

---

## How it works in this stack

VyOS runs as `site01` on the Switch-WAN segment (`192.168.122.0/24`, eth0) with a second interface on Switch-Site (`172.16.10.0/24`, eth1 gateway `172.16.10.1`). It provides:

- WAN connectivity for the Site-LAN via NAT
- Routing for sitepc01 (when deployed) toward the internet

The site branch is treated as an untrusted location — equivalent to a café network. All security enforcement happens on pop01, not at the site. sitepc01 will use a NetBird client for identity-based connectivity, identical to mobile01. VyOS provides the WAN uplink only — no security policy is applied at the branch.

**SD-WAN equivalent:** This mirrors Zscaler's Zero Trust SD-WAN architecture where the branch appliance provides connectivity while the Zero Trust Exchange provides inspection. No IPsec tunnels exist between site01 and pop01 — sitepc01 reaches resources exclusively via the NetBird WireGuard overlay.

**IP addressing:**
- eth0 (WAN): `192.168.122.33` (gateway `192.168.122.1`)
- eth1 (Site-LAN): `172.16.10.1/24`
- External access via GNS3 host: SSH port-forward to `10.158.10.67:7033`

**Hosts entry for NetBird:** VyOS resolves `netbird.sandbox.local` via static host mapping:
```
set system static-host-mapping host-name netbird.sandbox.local inet 192.168.122.23
```

---

## Known status

sitepc01 is cabled on Switch-Site (`172.16.10.50/24`) but has no OS installed. The VyOS gateway and Site-LAN are topologically complete; the endpoint node is pending OS installation and NetBird enrollment.

---

## Related

- [Architecture overview](../overview/architecture.md)
- [Component: GNS3](gns3.md)
- [Component: NetBird](netbird.md)
- [Concept: SASE](../concepts/sase.md)
- [Decision: GNS3 vs EVE-NG](../decisions/gns3-vs-eveng.md)

---
title: "Intune Endpoint Enforcement"
tags: [intune, mdm, firewall, identity]
---

# Intune Endpoint Enforcement

**Role:** The MDM that pushes the SASE posture onto managed Windows endpoints — the device-side complement to the network-side controls (Squid SWG, Unbound RPZ, NetBird overlay).  
**Platform:** Microsoft Intune / Entra ID (`aplab.be` tenant)  
**Config location:** Intune portal (`https://intune.microsoft.com`) — Configuration profiles, Endpoint Security Firewall, and Remediations, all scoped to the dynamic device group `2ITcsc1A-SASE-Devices`

---

## How it works in this stack

The managed Windows devices are Entra-joined and Intune-enrolled (see [Decision: Managed Windows Devices Scope](../decisions/managed-devices-scope.md)). That enrollment is what makes endpoint enforcement possible at all: instead of hand-configuring each client, Intune pushes the proxy, certificate, firewall, QoS, and routing config directly to the device.

The split between Intune and the rest of the stack maps onto the three-gate model (see [Decision: CA + NetBird Posture (Three-Gate Model)](../decisions/ca-posture-hybrid.md)). Entra ID Conditional Access evaluates identity (Gate 1) and consumes Intune's device-compliance attestation (Gate 2); this page is the *endpoint-config* side of that same managed-device foundation. Each profile enforces one piece of the SASE posture:

- **Forced PAC proxy** (`2ITCSC1A-SASE-Proxy-PAC`) routes browser traffic through the pop01 Squid SWG, so the proxy is actually forced on the device rather than a setting the user can clear.
- **Trusted-cert profile** (`2ITCSC1A-SASE-PoC-CA-Root`) installs the SASE-PoC-CA root so the SSL-Bump certificate chain is trusted and HTTPS does not break.
- **Firewall block-set** (`2ITCSC1A-SASE-FW-BypassBlock`) closes the two protocol-level bypass vectors (QUIC and DoT) that would otherwise route traffic around the SWG and the filtered resolver.
- **DSCP/QoS profile** (`2ITCSC1A-SASE-QoS-Teams`) carries the Teams traffic markings that the VyOS SASE gateway shapes on.
- **Route remediation** (`2ITCSC1A-Route-Remediation`) installs the M365 Optimize split-tunnel so latency-sensitive Microsoft media bypasses the NetBird exit node.

A recurring theme across all five deliverables: Intune's per-profile status field is *not* ground truth. The device store (`certutil -store`, `Get-ChildItem Cert:\LocalMachine\Root`, the registry, `Find-NetRoute`) is what was actually verified — a profile can read "Succeeded" while nothing landed (V41).

## Configuration

### Forced PAC proxy — `2ITCSC1A-SASE-Proxy-PAC`

A Custom OMA-URI profile driving the NetworkProxy CSP (the Settings Catalog does not surface NetworkProxy, so the surgical OMA-URI route is used):

```
./Vendor/MSFT/NetworkProxy/ProxySettingsPerUser   Integer  0
./Vendor/MSFT/NetworkProxy/AutoDetect             Integer  0
./Vendor/MSFT/NetworkProxy/SetupScriptUrl         String   http://wpad.sandbox.local/wpad.dat
```

`ProxySettingsPerUser=0` makes the setting system-wide (all WinINET-respecting apps), `AutoDetect=0` forces the explicit PAC instead of WPAD discovery, and `SetupScriptUrl` points at the existing WPAD/PAC target. This is the MDM enforcement of the SWG; the manual Windows-proxy method is the fallback ([Runbook 03: Proxy & WPAD](../runbooks/03-proxy-wpad.md), Step 8). No ProxyServer node is set on purpose — a PAC URL combined with a non-blank manual proxy server is a known Intune bug that breaks the profile. A reboot is required to clear the cert/proxy cache after the profile lands.

### Trusted-cert profile — `2ITCSC1A-SASE-PoC-CA-Root`

A Trusted-certificate template (Windows 10 and later) that uploads the public SASE-PoC-CA `.crt` (no private key) into the **Computer certificate store - Root**. This is what makes the SSL-Bump certificate (issuer `O=SASE PoC`, thumbprint `55EF83FE8BE080B7DF9AE92E9E55CFCEF3AC4537`) trusted on the device, so bumped HTTPS validates instead of throwing certificate errors.

### Firewall block-set — `2ITCSC1A-SASE-FW-BypassBlock`

Two targeted Microsoft Defender Firewall outbound rules via Endpoint Security → Firewall:

| Rule | Direction | Protocol | Remote port | Action | Effect |
|------|-----------|----------|-------------|--------|--------|
| `Block-QUIC-Outbound-UDP443` | Out | UDP (17) | 443 | Blocked | Forces browsers off QUIC back to TCP/443 so Squid can SSL-bump the session |
| `Block-DoT-Outbound-TCP853` | Out | TCP (6) | 853 | Blocked | Forces DNS through the filtered resolver (protects the RPZ / threat-intel path) |

This is deliberately a **targeted block-set, not a default-deny**. A full default-deny outbound bricks the MDM check-in path (the WinHTTP SYSTEM channel to the Microsoft control plane is a large CDN-backed FQDN set that IP-based firewall rules cannot allowlist cleanly) — exactly the IT1214934 footgun. DoH over TCP/443 is not port-blockable and stays a separate concern. The schema requires the TCP/UDP protocol to be set on any rule that has port ranges, otherwise the rule fails with "the parameter is incorrect".

### DSCP / QoS profile — `2ITCSC1A-SASE-QoS-Teams`

Three NetworkQoSPolicy-CSP nodes (officially supported on Intune-managed + Entra-joined devices), matching `ms-teams.exe` on source-port ranges and marking DSCP:

| Stream | Source ports | DSCP | Class |
|--------|--------------|------|-------|
| Teams audio | 50000–50019 | 46 | EF |
| Teams video | 50020–50039 | 34 | AF41 |
| Teams screenshare | 50040–50059 | 18 | AF21 |

The markings are validated on the endpoint by reading `HKLM:\SOFTWARE\Policies\Microsoft\Windows\QoS` (all three policies present, `NetProfile 7` = all profiles, `Protocol 3` = TCP+UDP). The marking-on-the-wire check runs on the VyOS path, where the SASE gateway sees the DSCP value downstream.

### Split-tunnel route remediation — `2ITCSC1A-Route-Remediation`

An Intune Remediation (detect + remediate, **runs as SYSTEM**, 64-bit, hourly) that adds the M365 Optimize routes pointing at the physical gateway instead of the NetBird `wt0` interface:

```
13.107.64.0/18
52.112.0.0/14
52.120.0.0/14
```

Windows picks the route on longest-prefix match, so a `/14` wins over NetBird's `/1` split regardless of metric. NetBird has no native route-exclusion, so an OS route via the Remediation is the only way to keep Optimize media off the exit node; SYSTEM context holds the route privilege, so it does not need the local-admin account. The remediate and detect scripts share the exact same prefix list, and the remediate script discovers the physical gateway dynamically (the `0.0.0.0/0` route on a non-`wt0` interface) rather than hardcoding it.

The Remediation decouples its exit code from the script's success message: a `try/catch` keeps and logs any `New-NetRoute` error (`-ErrorAction Stop`), and a verify-loop only allows exit 0 once every prefix actually resolves off-tunnel. Without that, a non-terminating `New-NetRoute` error walked silently into the success line and Intune logged exit 0 while no route existed — the same class of false "success" as the certificate `Succeeded` that lied. After a tick the device reports `Without issues` in the portal.

### Device scoping — `2ITcsc1A-SASE-Devices`

All of the above assign to one dynamic device group with the membership rule:

```
device.displayName -startsWith "2ITcsc1A-"
```

User-group targeting is too broad here — a Proxy or Firewall CSP on a classmate's device that is not on the overlay would break their internet — so config goes to the device group instead. The rule reads the Entra `displayName`, not the Intune-shown name. The prefix casing in the rule (`2ITcsc1A-`) differs from the all-caps profile names (`2ITCSC1A-…`) and from the device names (`2ITCSC1A-MOB-1`, `2ITCSC1A-SITE01`); that mismatch is harmless because string operations in dynamic rules are case-insensitive (only property names are case-sensitive).

## Integration points

| Interface | Direction | Details |
|-----------|-----------|---------|
| NetworkProxy CSP → WinINET | Outbound (config push) | Sets the system PAC URL that Edge/Chrome read, steering traffic to pop01 Squid `100.70.154.79:3128`; the WinHTTP SYSTEM proxy stays Direct by design for MDM check-in |
| Trusted-cert → Computer-Root store | Outbound (config push) | Installs SASE-PoC-CA so Squid SSL-Bump (issuer `O=SASE PoC`) is trusted |
| Endpoint Security Firewall → Defender Firewall | Outbound (config push) | Two outbound block rules (UDP/443 QUIC, TCP/853 DoT) |
| NetworkQoSPolicy CSP → registry | Outbound (config push) | Teams DSCP markings the VyOS SASE gateway shapes on |
| Route Remediation → Windows route table | Outbound (SYSTEM, hourly) | M365 Optimize split-tunnel off the NetBird exit node |
| Entra ID Conditional Access | Upstream (attestation consumer) | Gate 2 reads Intune device compliance; CA Policy 5 requires a compliant device. See [Runbook 07: Access Policy](../runbooks/07-access-policy.md) |

## Known issues / gotchas

- **Intune status lies on a shared CSP setting.** On the shared `aplab.be` tenant, a loosely-scoped Trusted-Root profile from another group targeting the same `Windows81TrustedRootCertificate` CSP setting silently blocks yours — both profiles report "Succeeded" while only one certificate actually lands in the store. The device store is the only ground truth. Resolution was to scope the conflicting profile so the managed device fell outside it; after sync the SASE-PoC-CA landed (V41).
- **IT1214934 — firewall reconciliation can brick the device.** Renaming or adding a firewall policy can flip an existing outbound rule into "block any outbound", cutting all internet communication with no remote control; the only recovery is a local login plus cleaning the `FirewallPolicy\Mdm\FirewallRules` registry. This is why the block-set is targeted, not default-deny, and why the local admin `.\saseuser` is a recovery prerequisite before the first firewall push. Bare `saseuser` fails (UAC reads it as an Entra account); the `.\saseuser` local-account prefix works.
- **`New-NetRoute` rejects `PersistentStore` on this build** with System Error 87 (`ERROR_INVALID_PARAMETER`) — it only writes ActiveStore. Persistence therefore comes from the hourly Remediation (self-healing against reboot and against a NetBird reconnect rewriting the table), not from a persistent route.
- **`tracert` is blind on the Optimize relays.** Microsoft Optimize relays do not answer ICMP/UDP probes, so `tracert` showing three stars does not mean the split-tunnel failed. Verify with `Find-NetRoute -RemoteIPAddress 52.112.0.1`, which asks Windows directly which interface it would choose (it returns `Ethernet0`, the physical gateway).
- **Enrollment closes the manual proxy.** Entra enrollment turns off the manually-configured PAC setup, so HTTPS goes Direct until the forced-PAC profile lands — which is precisely what that profile is for, not a bug.

## Related

- [Decision: CA + NetBird Posture (Three-Gate Model)](../decisions/ca-posture-hybrid.md)
- [Decision: Managed Windows Devices Scope](../decisions/managed-devices-scope.md)
- [Component: NetBird](netbird.md)
- [Component: Squid](squid.md)
- [Runbook 07: Access Policy](../runbooks/07-access-policy.md)
- [Runbook 03: Proxy & WPAD](../runbooks/03-proxy-wpad.md)

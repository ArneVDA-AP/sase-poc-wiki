---
title: "Runbook: Intune Endpoint"
tags: [runbook, intune, mdm, firewall, identity]
---

# Runbook: Intune Endpoint

**Source:** `Verslag41.md` + Route-Remediation addendum (3 June 2026); `Verslag44.md` (SITE01 enrollment)
**Node(s):** Entra ID / Intune (`aplab.be` tenant) → managed Windows endpoints; pop01 Squid (verification target)
**Prerequisites:** Full SASE stack operational (Runbooks 01–07); a managed Windows device Entra-joined + Intune-enrolled
**Status:** Implemented (Verslag41) — five profiles built and verified on `2ITCSC1A-MOB-1`; second device `2ITCSC1A-SITE01` enrolled (Verslag44).

> This runbook is the **deployment trail** for the device-side SASE posture: it pushes the certificate, forced proxy, firewall, QoS, and split-tunnel routes onto managed Windows endpoints through Intune. The config detail and the *why* behind each profile live in [Component: Intune Endpoint Enforcement](../components/intune-endpoint-enforcement.md) — this page gives the steps and the verification that each piece actually landed. A recurring rule throughout: **the device store is ground truth, not the Intune per-profile status field** (a profile can read "Succeeded" while nothing landed).

---

## Prerequisites checklist

- [ ] Entra ID tenant (`aplab.be`) accessible with Intune Administrator rights
- [ ] Target device Entra-joined + Intune-enrolled (verified via `dsregcmd /status` → `AzureAdJoined: YES`, `Managed by MDM`)
- [ ] The licensed enrollment account is the device's primary user (`student_1`, A5 → Intune Plan 1); the persona account on the overlay may differ — device-config profiles target the device group on `displayName`, not the user
- [ ] Local admin `.\saseuser` prepared as a recovery path **before the first firewall push** (see Step 1)
- [ ] Squid overlay listener up on pop01 (`100.70.154.79:3128`) so the forced-PAC and bump can be verified
- [ ] Microsoft control-plane endpoints on the Squid splice/no-bump list (Runbook 03) so MDM check-in and registration are not bumped
- [ ] "Remediations" enabled in Intune (clears the one-time licensing-data prerequisite that greys out **Create**)

> **Gotcha: prepare `.\saseuser` before the firewall push.** A firewall reconciliation bug (IT1214934) can flip an existing outbound rule into "block any outbound" and cut all internet with no remote control — recovery is a local login plus cleaning the `FirewallPolicy\Mdm\FirewallRules` registry. Bare `saseuser` fails at the UAC prompt (it is read as an Entra account); the local-account prefix `.\saseuser` works. Confirm the recovery login *before* Step 4.

---

## Step 1: Create the dynamic device group

Config goes to a **device** group, not a user group — a Proxy or Firewall CSP on a classmate's device that is not on the overlay would break their internet.

Navigate to: `https://intune.microsoft.com → Groups → New group → Security → Dynamic Device`

```
Name: 2ITcsc1A-SASE-Devices
Membership rule:  device.displayName -startsWith "2ITcsc1A-"
```

The rule reads the **Entra** `displayName`, not the Intune-shown name. The casing in the rule (`2ITcsc1A-`) differs from the all-caps device names (`2ITCSC1A-MOB-1`, `2ITCSC1A-SITE01`); that is harmless because string operations in dynamic rules are case-insensitive (only property names are case-sensitive).

**Verify:** Group → **Validate rules** tab passes for the device name; Group → Members lists the target device.

---

## Step 2: Push the trusted-cert profile (`2ITCSC1A-SASE-PoC-CA-Root`)

Installs the SASE-PoC-CA root so the Squid SSL-Bump certificate chain is trusted and bumped HTTPS validates instead of throwing certificate errors.

On pop01 (OPNsense): `System → Trust → Authorities`, export the **public** `.crt` next to `SASE-PoC-CA` (no `.p12` / private key).

Navigate to: `Intune → Devices → Configuration → Windows 10 and later → Templates → Trusted certificate`

```
Name: 2ITCSC1A-SASE-PoC-CA-Root
Certificate file: SASE-PoC-CA public .crt
Destination store: Computer certificate store - Root
Assignment: 2ITcsc1A-SASE-Devices
```

**Verify in the device store, not the Intune status:**

```powershell
certutil -store Root | findstr /i SASE
# Issuer/Subject: O=SASE PoC ; Thumbprint 55EF83FE8BE080B7DF9AE92E9E55CFCEF3AC4537
Get-ChildItem Cert:\LocalMachine\Root | Where-Object Subject -match 'SASE PoC'
```

> **Gotcha: shared-tenant CSP conflict.** On the shared `aplab.be` tenant a loosely-scoped Trusted-Root profile from another group targeting the same `Windows81TrustedRootCertificate` CSP setting silently blocks yours — both profiles report "Succeeded" while only one certificate lands. If `findstr SASE` returns nothing but another group's CA is present, scope the conflicting profile so the device falls outside it; after sync the SASE-PoC-CA lands. The device store is the only ground truth.

---

## Step 3: Push the forced-PAC proxy profile (`2ITCSC1A-SASE-Proxy-PAC`)

Forces browser traffic through the pop01 Squid SWG so the proxy is enforced on the device, not a setting the user can clear. The Settings Catalog does not surface NetworkProxy, so a Custom OMA-URI profile drives the NetworkProxy CSP directly.

Navigate to: `Intune → Devices → Configuration → Windows 10 and later → Templates → Custom → Add OMA-URI Settings`

```
./Vendor/MSFT/NetworkProxy/ProxySettingsPerUser   Integer  0
./Vendor/MSFT/NetworkProxy/AutoDetect             Integer  0
./Vendor/MSFT/NetworkProxy/SetupScriptUrl         String   http://wpad.sandbox.local/wpad.dat
```

`ProxySettingsPerUser=0` makes it system-wide (all WinINET apps), `AutoDetect=0` forces the explicit PAC. Set **no** `ProxyServer` node — a PAC URL combined with a non-blank manual proxy server is a known Intune bug that breaks the profile. Name `2ITCSC1A-SASE-Proxy-PAC`, assign to `2ITcsc1A-SASE-Devices`. **Reboot** to clear the cert/proxy cache after the profile lands.

**Verify traffic goes through Squid:** browse `https://google.com` — the certificate issuer is `O=SASE PoC` (bumped, the device trusts the pushed CA). On pop01, follow the Squid access log: the device's overlay source IP appears against `TCP_MISS`/`TCP_TUNNEL` lines (no collapse to a gateway IP), confirming the traffic is proxied.

> **Note:** Entra enrollment turns off the manually-configured PAC, so HTTPS goes Direct (a "white page") until this profile lands — that is exactly what the profile is for, not a bug. The WinHTTP SYSTEM proxy stays Direct by design for MDM check-in.

---

## Step 4: Push the firewall block-set (`2ITCSC1A-SASE-FW-BypassBlock`)

Closes the two protocol-level bypass vectors (QUIC and DoT) that would otherwise route traffic around the SWG and the filtered resolver. Confirm the `.\saseuser` recovery login (prerequisite) before this push.

Navigate to: `Intune → Endpoint security → Firewall → Create policy → Windows → Microsoft Defender Firewall Rules`

```
Name: 2ITCSC1A-SASE-FW-BypassBlock

Rule: Block-QUIC-Outbound-UDP443
  Direction: Out | Protocol: UDP (17) | Remote ports: 443 | Action: Blocked
Rule: Block-DoT-Outbound-TCP853
  Direction: Out | Protocol: TCP (6)  | Remote ports: 853 | Action: Blocked

Assignment: 2ITcsc1A-SASE-Devices
```

This is deliberately a **targeted block-set, NOT a default-deny**. A full default-deny outbound bricks the MDM check-in path (the WinHTTP SYSTEM channel is a large CDN-backed FQDN set that IP-based rules cannot allowlist cleanly) — the IT1214934 footgun. The schema requires the TCP/UDP protocol to be set on any rule that has port ranges, otherwise the rule fails with "the parameter is incorrect".

**Verify nothing legitimate breaks:**

- `netbird status` → Connected (the UDP/443 block does not touch the WireGuard tunnel on 51820)
- `https://google.com` loads, issuer `O=SASE PoC` (browsers fall back from QUIC to TCP/443, Squid bumps it)
- Intune → Devices → device → Sync → **successful** (the WinHTTP SYSTEM path stays open)

---

## Step 5: Push the DSCP/QoS profile (`2ITCSC1A-SASE-QoS-Teams`)

Carries the Teams traffic markings that the VyOS SASE gateway shapes on. The NetworkQoSPolicy CSP is officially supported on Intune-managed + Entra-joined devices (the device's exact state).

Navigate to: `Intune → Devices → Configuration → Windows 10 and later → Templates → Custom → Add OMA-URI Settings` (three nodes per stream, prefix `./Device/Vendor/MSFT/NetworkQoSPolicy/`):

| Stream | `SourcePortMatchCondition` | `DSCPAction` | `AppPathNameMatchCondition` |
|--------|----------------------------|--------------|-----------------------------|
| TeamsAudio | `50000-50019` | `46` | `ms-teams.exe` |
| TeamsVideo | `50020-50039` | `34` | `ms-teams.exe` |
| TeamsScreenshare | `50040-50059` | `18` | `ms-teams.exe` |

Name `2ITCSC1A-SASE-QoS-Teams`, assign to `2ITcsc1A-SASE-Devices`.

**Verify the registry markings** (readable as a non-admin user, since HKLM read does not need elevation):

```powershell
Get-ChildItem 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\QoS'
# TeamsAudio / TeamsVideo / TeamsScreenshare each present:
#   DSCP 46/34/18, NetProfile 7 (all profiles), Protocol 3 (TCP+UDP)
```

> **Note:** The marking-on-the-wire check belongs on the VyOS path (the SASE gateway reads the DSCP value downstream); a local outbound capture on the endpoint reports the mark unreliably because it sits under the NDIS hook.

---

## Step 6: Deploy the split-tunnel route Remediation (`2ITCSC1A-Route-Remediation`)

Adds the M365 Optimize routes pointing at the physical gateway instead of the NetBird `wt0` interface, so latency-sensitive Microsoft media bypasses the exit node. NetBird has no native route-exclusion, so an OS route via a Remediation is the only way to keep Optimize media off the tunnel. Windows picks the route on longest-prefix match, so a `/14` wins over NetBird's `/1` split regardless of metric.

Navigate to: `Intune → Devices → Remediations → Create script package`

```
Name: 2ITCSC1A-Route-Remediation
Run this script using the logged-on credentials: No   (runs as SYSTEM)
Enforce script signature check: No
Run script in 64-bit PowerShell: Yes
Schedule: Hourly
Assignment: 2ITcsc1A-SASE-Devices
```

Prefix list shared by both scripts (detect + remediate must use the **exact same** list):

```
13.107.64.0/18
52.112.0.0/14
52.120.0.0/14
```

The detect script reports the routes missing (exit 1) when any prefix is absent off-tunnel; the remediate script discovers the physical gateway dynamically (the `0.0.0.0/0` route on a non-`wt0` interface) and re-adds the routes. SYSTEM holds the route privilege, so this does not need the local-admin account.

**Apply the addendum fix — decouple the exit code from the state.** The remediate script must:

1. set `-ErrorAction Stop` on `New-NetRoute` so a failed add becomes a terminating error;
2. wrap it in a `try/catch` that **keeps and logs** the error instead of swallowing it;
3. run a **verify-loop** that allows exit 0 only once every prefix actually resolves off-tunnel (`InterfaceAlias -ne 'wt0'`).

Without this, a non-terminating `New-NetRoute` error walks silently into the success line and Intune logs exit 0 while no route exists — the same class of false "success" as the certificate `Succeeded` that lied.

**Verify:**

```powershell
Find-NetRoute -RemoteIPAddress 52.112.0.1
# InterfaceAlias: Ethernet0 (the physical gateway) — not wt0
```

After a tick the device reports **Without issues** in the portal.

> **Gotcha: `New-NetRoute` rejects `PersistentStore` on this build** (System Error 87, `ERROR_INVALID_PARAMETER`) — it only writes ActiveStore. Persistence therefore comes from the hourly Remediation (self-healing against reboot and against a NetBird reconnect rewriting the table), not from a persistent route.
>
> **Gotcha: `tracert` is blind on the Optimize relays.** Microsoft Optimize relays do not answer ICMP/UDP probes, so `tracert` showing three stars does not mean the split-tunnel failed. Verify with `Find-NetRoute`, which asks Windows directly which interface it would choose.

---

## Step 7: Worked enrollment example — SITE01 (Tiny11) reaches Entra-joined + Intune-enrolled

The remote-site machine `2ITCSC1A-SITE01` (a Tiny11 image) was brought to Entra-joined + Intune-enrolled with `student_1`. This is the working procedure for enrolling a second managed device so the profiles above land on it too.

Procedure (all join actions **at the console**, not over RDP — the WAM flow behaves differently in a remote session):

1. **Account:** enroll with the licensed account `student_1` (the only Intune-licensed account; the persona account `docent1` is deliberately unlicensed). The persona layer (NetBird) and the device-owner layer (Entra) are independent.
2. **Enable the Workplace Join task set** (a Tiny11 debloat disables them, which produces `0x80041326 SCHED_E_TASK_DISABLED` on join):
   ```powershell
   Get-ScheduledTask -TaskPath "\Microsoft\Windows\Workplace Join\" | Enable-ScheduledTask
   ```
   Enable all three (Device-Sync carries the periodic CSP pull — enabling only the join task would later leave the device joined but pulling no policies).
3. **Start the Sign-in Assistant** (Tiny11 leaves it stopped):
   ```powershell
   Set-Service wlidsvc -StartupType Manual; Start-Service wlidsvc
   ```
4. **Confirm the Microsoft auth chain splices on the proxy.** Interactive auth over the PAC needs the full no-bump set; ensure `.microsoftazuread-sso.com` and `.live.com` are on the Squid splice list alongside `.microsoftonline.com` and `enterpriseregistration.windows.net` (Runbook 03).
5. **Use the Entra *join*, not registration.** The email-field flow does a device *registration*; a registered device does not fall in the dynamic device group, so the profiles never land. If `dsregcmd` shows `WorkplaceJoined: YES` / `AzureAdJoined: NO`, disconnect that account, reboot, and use the "Join this device to Microsoft Entra ID" link (an existing WorkplaceJoined state hides it).

**Result — enrolled:**

```powershell
dsregcmd /status
#   AzureAdJoined : YES
#   DeviceId      : c72bcdc0-12ce-4217-817d-ea105604b54a
#   Managed by MDM (auto-enroll, MDM scope All)
```

The `displayName 2ITCSC1A-SITE01` matches the dynamic rule, so the device enters `2ITcsc1A-SASE-Devices` and the cert/proxy/QoS profiles land on it. The device was independently confirmed against the device store (`dsregcmd`), bump/splice/RPZ all proven from SITE01.

**Remaining task on this image:** Defender is off on the Tiny11 build, so the device fails the antivirus compliance check ([Runbook 07: Access Policy](07-access-policy.md), Gate 2). Restore Defender before the compliance evaluation matters, or document the Defender-off state as a deliberate Tiny11 gap alongside the software-KSP TPM gap.

---

## Validation checklist

- [ ] `2ITcsc1A-SASE-Devices` dynamic group resolves the target device (Members tab)
- [ ] `2ITCSC1A-SASE-PoC-CA-Root` → `certutil -store Root` shows `O=SASE PoC` (thumbprint `55EF…`); no other group's CA present
- [ ] `2ITCSC1A-SASE-Proxy-PAC` → `https://google.com` issuer `O=SASE PoC`; overlay source IP in the pop01 access log
- [ ] `2ITCSC1A-SASE-FW-BypassBlock` → `netbird status` Connected, web via proxy works, Intune Sync successful (targeted block-set, not default-deny)
- [ ] `2ITCSC1A-SASE-QoS-Teams` → three QoS policies in `HKLM:\SOFTWARE\Policies\Microsoft\Windows\QoS` (DSCP 46/34/18, NetProfile 7, Protocol 3)
- [ ] `2ITCSC1A-Route-Remediation` → `Find-NetRoute -RemoteIPAddress 52.112.0.1` returns `Ethernet0`; portal shows **Without issues**
- [ ] `.\saseuser` recovery login confirmed before the firewall push
- [ ] Second device (SITE01) `AzureAdJoined: YES`, in the device group; Defender-off compliance item tracked

---

## Related

- [Component: Intune Endpoint Enforcement](../components/intune-endpoint-enforcement.md)
- [Component: NetBird](../components/netbird.md)
- [Component: Squid](../components/squid.md)
- [Decision: Managed Devices Scope](../decisions/managed-devices-scope.md)
- [Decision: CA + Posture Hybrid](../decisions/ca-posture-hybrid.md)
- [Runbook 07: Access Policy](07-access-policy.md)
- [Runbook 03: Proxy & WPAD](03-proxy-wpad.md)

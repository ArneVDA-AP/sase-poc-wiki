---
title: "Runbook: Access Policy"
tags: [runbook, entra-id, zero-trust, conditional-access, posture-check]
---

# Runbook: Access Policy

**Source:** `raw/Doc7_ZTNA_Context_Aware.md`
**Node(s):** Entra ID (`aplab.be` tenant) + NetBird Dashboard
**Prerequisites:** All prior runbooks completed (full SASE stack operational)
**Status:** Planned — architecture validated (Addendum E), Gates 1 & 2 not yet implemented. Gate 3 fully operational.

> **This runbook describes planned steps.** Gates 1 and 2 are architecture design and implementation plan, scheduled for post-interim evaluation (20 April 2026). Gate 3 is operational and covered by Runbooks 03-06. Status labels `[PLANNED]` and `[OPERATIONAL]` are absolute.

---

## Prerequisites checklist

- [ ] Entra ID tenant (`aplab.be`) accessible
- [ ] A5 license active (confirmed 1 April 2026) — includes Entra ID P1 + P2
- [ ] Conditional Access Administrator or Global Admin rights
- [ ] NetBird app registration (`cebe0d74-...`) present in Entra ID
- [ ] NetBird login via Microsoft working (Runbook 02, Step 8)
- [ ] Test account available: `arne.vda.2itcsc1a@aplab.be`
- [ ] **MFA registered for test account** (via `https://aka.ms/mfasetup`)
- [ ] NetBird version on mobile01 noted: `netbird version`
- [ ] NetBird Dashboard accessible: `https://netbird.sandbox.local`

> **Gotcha: MFA must be registered BEFORE enabling the CA MFA policy.** Unregistered MFA produces a "MFA required but not configured" block on the next login — including the verification session itself. Register MFA first, then enable the policy.

---

## [PLANNED] Step 1: Create Entra ID Conditional Access policies

Navigate to: `https://entra.microsoft.com → Protection → Conditional Access → Policies → New policy`

### Policy 1 — SASE-PoC-MFA-Required

```
Name: SASE-PoC-MFA-Required

Assignments:
  Users: All users
  Exclude: (optional) break-glass admin account

  Target resources:
    Cloud apps → Select apps → "NetBird SASE PoC"
    (Client ID: cebe0d74-be9f-49ac-9f35-65f11586c1bb)

  Conditions:
    Client apps → Configure: Yes
      checked: Browser
      checked: Mobile apps and desktop clients

Access controls:
  Grant: Require multifactor authentication

Enable policy: Report-only (first test) → then On
```

### Policy 2 — SASE-PoC-Geo-Block

First create a Named Location:

```
Protection → Conditional Access → Named locations → + Countries location
Name: SASE-PoC-Allowed-Countries
checked: Belgium
checked: Netherlands (optional — for commuting students)
```

Then create the policy:

```
Name: SASE-PoC-Geo-Block

Assignments:
  Users: All users
  Target resources: Select apps → "NetBird SASE PoC"

  Conditions:
    Locations → Configure: Yes
      Include: Any location
      Exclude: SASE-PoC-Allowed-Countries

Access controls:
  Grant: Block access

Enable policy: On
```

### Policy 3 — SASE-PoC-Block-Legacy-Auth

```
Name: SASE-PoC-Block-Legacy-Auth

Assignments:
  Users: All users
  Target resources: Select apps → "NetBird SASE PoC"

  Conditions:
    Client apps → Configure: Yes
      checked: Exchange ActiveSync clients
      checked: Other clients
      (DO NOT check Browser or Mobile apps)

Access controls:
  Grant: Block access

Enable policy: On
```

### Policy 4 — SASE-PoC-Risk-Block (requires A5/P2)

```
Name: SASE-PoC-Risk-Block

Assignments:
  Users: All users
  Target resources: Select apps → "NetBird SASE PoC"

  Conditions:
    Sign-in risk → Configure: Yes
      checked: High
      checked: Medium

Access controls:
  Grant: Require multifactor authentication
  (For High risk: consider Block access)

Enable policy: On
```

**Checkpoint Gate 1:**

- [ ] All 4 policies target the NetBird app registration
- [ ] MFA registration complete for test account
- [ ] Test login on mobile01: MFA prompt appears
- [ ] Entra ID → Protection → Sign-in logs → Conditional Access tab: policies show "Success"

---

## [PLANNED] Step 2: Create NetBird Posture Check

NetBird Dashboard → Access Control → Posture Checks → Create Posture Check

**Name:** `SASE-PoC-Compliance`
**Description:** Hybrid posture check — OS version, AV process, geo, client version

### Check 1 — OS Version

```
Windows: minimum kernel version 10.0.19041
```

> **Gotcha: This is the kernel version, not the marketing version.** Windows 10 "21H1" vs kernel 10.0.19041 — use the kernel version. 10.0.19041 is Windows 10 2004, the first version with native WireGuard kernel module.

### Check 2 — NetBird Client Version

```
Minimum version: <fill in output of 'netbird version' on mobile01>
```

### Check 3 — Process Check (Antivirus)

```
Windows path: C:\Program Files\Windows Defender\MsMpEng.exe
macOS path:   /usr/libexec/syspolicyd  (XProtect)
Linux path:   /usr/sbin/clamd          (ClamAV daemon)
```

> **Gotcha: If no path is specified for a given OS, NetBird blocks connections from that OS by default.** iOS and Android have no `process_check` available — mobile platforms are blocked if the posture check includes a process check. For the PoC this is acceptable (mobile01 is Windows). Note that process checks are spoofable — a dummy binary at the expected path satisfies the check without actually running the security software.

### Check 4 — Geo-location Check

```
Action:    Allow
Countries: Belgium
```

This is defense-in-depth alongside the CA geo-block (different GeoIP databases mitigate each other's errors). See [Decision: CA + Posture hybrid](../decisions/ca-posture-hybrid.md).

---

## [PLANNED] Step 3: Link posture check to ACL policies

```
NetBird Dashboard → Access Control → Policies
→ Select each relevant policy (Mobile-to-Services, Datacenter Access)
→ Tab: Posture Checks
→ Browse Checks → select "SASE-PoC-Compliance"
→ Add Posture Checks
→ Save Changes
```

Posture checks are per-policy, not global. Link to every policy that BYOD clients use.

**Checkpoint Gate 2:**

- [ ] Posture check "SASE-PoC-Compliance" created with all 4 checks
- [ ] Linked to Mobile-to-Services and Datacenter Access policies
- [ ] mobile01 connects: `netbird status` → Connected
- [ ] Dashboard → Peers → mobile01 → Posture Checks shows "Passed"

---

## [PLANNED] Step 4: Validation scenarios

### Scenario 1 — Positive test: compliant device, correct location

| Step | Action | Expected |
|------|--------|----------|
| 1 | `netbird down` on mobile01 | Tunnel closed |
| 2 | `netbird up` | Microsoft login page opens |
| 3 | Login + MFA | MFA prompt, login succeeds |
| 4 | Wait for tunnel | `netbird status` → Connected |
| 5 | `ping 100.70.154.79` | Response received |
| 6 | Check sign-in log | CA policies: "Success" |

### Scenario 2 — Negative test Gate 1: geo-block

| Step | Action | Expected |
|------|--------|----------|
| 1 | Remove Belgium from SASE-PoC-Allowed-Countries | Policy now blocks school IP |
| 2 | `netbird down && netbird up` | Login page opens |
| 3 | Login | "Access denied" — CA blocks |
| 4 | `netbird status` | Not Connected |
| 5 | Sign-in log | SASE-PoC-Geo-Block → "Failure" |
| 6 | **Restore:** add Belgium back | Next login succeeds |

### Scenario 3 — Negative test Gate 2: OS version

| Step | Action | Expected |
|------|--------|----------|
| 1 | Change posture check: Windows min kernel `10.0.99999` | Impossibly high version |
| 2 | `netbird down && netbird up` | Gate 1 passes (login OK) |
| 3 | `netbird status` | Authenticated but routes unavailable |
| 4 | **Restore:** set kernel back to `10.0.19041` | Connection restored |

### Scenario 4 — Negative test Gate 2: process check

| Step | Action | Expected |
|------|--------|----------|
| 1 | Stop Defender: `Set-MpPreference -DisableRealtimeMonitoring $true` (as admin) | MsMpEng.exe stops |
| 2 | `netbird down && netbird up` | Gate 1 passes |
| 3 | `netbird status` | Routes unavailable |
| 4 | **Restore:** `Set-MpPreference -DisableRealtimeMonitoring $false` | AV restarts, posture passes |

> Note: Tamper Protection may need to be disabled first: Settings → Windows Security → Virus & Threat Protection → Tamper Protection: Off. Re-enable immediately after testing.

### Scenario 5 — End-to-end: all three gates

| Step | Gate | Action | Expected |
|------|------|--------|----------|
| 1 | — | `netbird up` | Login page opens |
| 2 | Gate 1 | Login + MFA | Token received |
| 3 | Gate 2 | NetBird evaluates posture | OS OK, AV running, geo OK → tunnel active |
| 4 | Gate 3 | Browse `https://google.com` | SSL Bump: cert from SASE-PoC-CA |
| 5 | Gate 3 | Download EICAR test file | ClamAV blocks |
| 6 | Gate 3 | Browse RPZ-blocked domain | DNS NXDOMAIN |
| 7 | — | Sign-in log | CA policies: "Success" |
| 8 | — | Dashboard → peer info | Posture check: "Passed" |

---

## Honest limitations

| Limitation | Why | Production solution |
|-----------|-----|---------------------|
| Process check is spoofable | Binary on correct path ≠ working AV | Defender for Endpoint attestation (TPM) |
| No disk encryption verification | NetBird process_check can't check this | Intune compliance policy (requires MDM) |
| No AV definition version check | Process check only verifies the process runs | Intune/Defender health attestation |
| CA device compliance unavailable | Unmanaged BYOD, no Intune enrollment | Intune enrollment (unrealistic for 4000 BYOD) |
| Posture evaluates at tunnel setup, not continuously | No real-time session evaluation | NetBird periodic re-evaluation; CAE for production |
| GeoIP not 100% accurate | IP geolocation databases have errors | Mitigated by combining CA geo-block (Gate 1) with NetBird geo-check (Gate 2) — different GeoIP databases |
| C2 beaconing via WireGuard tunnel | A compromised device can use the WireGuard tunnel (port 51820) as a C2 beaconing channel — outside Squid's visibility | Gate 2 (posture) + endpoint detection; Suricata sees WireGuard as encrypted UDP |

> **Coverage estimate:** The PoC implements ~60–70% of the context-aware access control that commercial SASE solutions (Zscaler ZPA, Netskope Private Access) provide. The main gaps are endpoint attestation (TPM-based, not process-based) and Continuous Access Evaluation (CAE).

See [Concept: Zero Trust](../concepts/zero-trust.md) for the full three-gate model analysis.

---

## Related

- [Component: NetBird](../components/netbird.md)
- [Concept: Zero Trust](../concepts/zero-trust.md)
- [Decision: CA + Posture hybrid (Three-Gate Model)](../decisions/ca-posture-hybrid.md)
- [Decision: Zitadel as IdP broker](../decisions/zitadel-idp-broker.md)

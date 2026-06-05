---
title: "Runbook: Access Policy"
tags: [runbook, entra-id, zero-trust, conditional-access, posture-check]
---

# Runbook: Access Policy

**Source:** `Verslag40.md` (implementation, 2 June 2026); `Doc7_ZTNA_Context_Aware.md` + Addendum E.2 (design intent)
**Node(s):** Entra ID (`aplab.be` tenant) + Intune + NetBird Dashboard
**Prerequisites:** All prior runbooks completed (full SASE stack operational)
**Status:** Implemented (Verslag40) — Gate 1: 5 CA policies (3 On, 2 Report-only); Gate 2: Intune device compliance operational; Gate 3 fully operational.

> **Gates 1 and 2 are implemented (Verslag40).** Gate 1 = five Conditional Access policies: MFA Required (On), Block Legacy Auth (On), Risk-Based Block (On), Geo-Block Belgium only (Report-only), Require Compliant Device (Report-only → "Report-only: Success"). The two Report-only policies stay that way until demo preparation. Gate 2 = Intune device compliance (`2ITCSC1A-SASE-Windows-Compliance`), attestation-based — this is the deployed device gate, **not** NetBird posture checks (which remain an optional, undeployed defense-in-depth layer). Gate 3 is operational and covered by Runbooks 03-06.

---

## Prerequisites checklist

- [ ] Entra ID tenant (`aplab.be`) accessible
- [ ] A5 license active (confirmed 1 April 2026) — includes Entra ID P1 + P2
- [ ] Conditional Access Administrator or Global Admin rights
- [ ] NetBird app registration `2ITCSC1A-Netbird-Sandbox` (App ID `11803ee8-eb15-462c-a286-5415c17a29c6`) present in Entra ID — for NetBird login only, **not** a CA target (see Step 1 gotcha)
- [ ] NetBird login via Microsoft working (Runbook 02, Step 8)
- [ ] Persona security groups present: `2ITCSC1A-Studenten`, `2ITCSC1A-Docenten`, `2ITCSC1A-Admins`
- [ ] Test accounts (single-persona): a studenten member (e.g. `Student_1@aplab.be`, A5-licensed so Intune applies), a docenten member, and `2itcsc1a_admin1` (admins / break-glass)
- [ ] **MFA registered for test account** (via `https://aka.ms/mfasetup`)
- [ ] mobile01 Entra-joined + Intune-enrolled (appears in Intune as `2ITCSC1A-MOB-1`)
- [ ] NetBird version on mobile01 noted: `netbird version`
- [ ] NetBird Dashboard accessible: `https://netbird.sandbox.local`

> **Gotcha: MFA must be registered BEFORE enabling the CA MFA policy.** Unregistered MFA produces a "MFA required but not configured" block on the next login — including the verification session itself. Register MFA first, then enable the policy.

---

## [IMPLEMENTED] Step 1: Create Entra ID Conditional Access policies (Gate 1)

Navigate to: `https://entra.microsoft.com → Protection → Conditional Access → Policies → New policy`

> **Gotcha: target All resources, NOT the NetBird app.** App-targeted CA policies never fire on NetBird/Zitadel OIDC sign-ins. CA matches on the token's *resource* — an OIDC login requests Microsoft Graph (`00000003-0000-0000-c000-000000000000`), not the NetBird app registration — so a policy scoped to "Select apps → NetBird" shows **Not Applied** on every NetBird login (Verslag40, B40.4). All five policies therefore target **All resources** and contain blast radius through **user-scoping** (persona groups as include) rather than app-scoping — the inverse of Addendum E's strategy.
>
> **No resource-exclusions.** From 15 June 2026 Microsoft enforces All-resources policies that carry resource-*exclusions* even on OIDC-only sign-ins; before that date such policies are not enforced for OIDC-only sign-ins (a bypass). All-resources *without* exclusions is enforced consistently on both sides of that date, so every policy uses All resources with no resource-exclusion (B40.5). Blast-radius control is purely via user-scoping: include the persona groups (`2ITCSC1A-Studenten`, `2ITCSC1A-Docenten`, `2ITCSC1A-Admins`), exclude `2itcsc1a_admin1` as break-glass on every policy. Because `2itcsc1a_admin1` is the only member of the admins persona and is excluded everywhere, the admins persona effectively falls under no CA policy (B40.6 — accepted for now).

### Policy 1 — 2ITCSC1A-SASE-PoC-MFA-Required

```
Name: 2ITCSC1A-SASE-PoC-MFA-Required

Assignments:
  Users: Include 2ITCSC1A-Studenten + 2ITCSC1A-Docenten + 2ITCSC1A-Admins
  Exclude: 2itcsc1a_admin1 (break-glass)

  Target resources: All resources (no app-targeting, no resource-exclusions)

  Conditions:
    Client apps → Configure: Yes
      checked: Browser
      checked: Mobile apps and desktop clients

Access controls:
  Grant: Require multifactor authentication

Enable policy: On
```

### Policy 2 — 2ITCSC1A-SASE-PoC-Geo-Block

First create a Named Location:

```
Protection → Conditional Access → Named locations → + Countries location
Name: 2ITCSC1A-SASE-PoC-Allowed-Countries
checked: Belgium
checked: Netherlands (optional — for commuting students)
```

Then create the policy:

```
Name: 2ITCSC1A-SASE-PoC-Geo-Block

Assignments:
  Users: Include persona groups / Exclude 2itcsc1a_admin1
  Target resources: All resources

  Conditions:
    Locations → Configure: Yes
      Include: Any location
      Exclude: 2ITCSC1A-SASE-PoC-Allowed-Countries

Access controls:
  Grant: Block access

Enable policy: Report-only (→ On at demo)
```

### Policy 3 — 2ITCSC1A-SASE-PoC-Block-Legacy-Auth

```
Name: 2ITCSC1A-SASE-PoC-Block-Legacy-Auth

Assignments:
  Users: Include persona groups / Exclude 2itcsc1a_admin1
  Target resources: All resources

  Conditions:
    Client apps → Configure: Yes
      checked: Exchange ActiveSync clients
      checked: Other clients
      (DO NOT check Browser or Mobile apps)

Access controls:
  Grant: Block access

Enable policy: On
```

### Policy 4 — 2ITCSC1A-SASE-PoC-Risk-Block (requires A5/P2)

```
Name: 2ITCSC1A-SASE-PoC-Risk-Block

Assignments:
  Users: Include persona groups / Exclude 2itcsc1a_admin1
  Target resources: All resources

  Conditions:
    Sign-in risk → Configure: Yes
      checked: High
      checked: Medium

Access controls:
  Grant: Require multifactor authentication
  (For High risk: consider Block access)

Enable policy: On
```

### Policy 5 — 2ITCSC1A-SASE-Require-Compliant-Device

This policy ties Gate 1 to Gate 2: it requires the device to be marked compliant by Intune (Step 2). Note the name has **no** "PoC" segment.

```
Name: 2ITCSC1A-SASE-Require-Compliant-Device

Assignments:
  Users: Include 2ITCSC1A-Studenten / Exclude 2itcsc1a_admin1
    (studenten-only — docent1/admin1 are deliberately unlicensed, so scoping
     studenten-only avoids a lockout — B40.20)
  Target resources: All resources

  Conditions:
    Device platforms → Configure: Yes → Windows

Access controls:
  Grant: Require device to be marked as compliant

Enable policy: Report-only (→ On at demo) — gave "Report-only: Success" on a compliant mobile01 login
```

**Checkpoint Gate 1:**

- [ ] All 5 policies target **All resources** (no app-targeting, no resource-exclusions)
- [ ] Each policy includes the persona groups and excludes `2itcsc1a_admin1`
- [ ] MFA registration complete for the test account
- [ ] Test login on mobile01: MFA prompt appears
- [ ] Entra ID → Protection → Sign-in logs → Conditional Access tab shows `Resource: Microsoft Graph — Matched` and policies as "Success" / "Report-only: Success"

---

## [IMPLEMENTED] Step 2: Create the Intune device compliance policy (Gate 2)

Gate 2 is **Intune device compliance**, not NetBird posture. Because the in-scope devices are managed (Entra-joined + Intune-enrolled), Intune attests actual device state through the management agent — posture the end user cannot spoof. Policy 5 (Step 1) consumes this attestation at authentication time.

Navigate to: `https://intune.microsoft.com → Devices → Compliance → Policies → Create policy → Windows 10 and later`

```
Name: 2ITCSC1A-SASE-Windows-Compliance
Platform: Windows 10 and later

Compliance settings:
  Microsoft Defender Antimalware:                                  Require
  Defender Antivirus:                                             Require
  Defender Antispyware:                                           Require
  Defender Antimalware security intelligence up-to-date:          Require
  Real-time protection:                                          Require
  Firewall:                                                      Require
  Minimum OS version:                                            10.0.22000.0
  BitLocker / Secure Boot / Code integrity / TPM / encryption:   Not configured

Actions for noncompliance:
  Mark device noncompliant: Immediately (0 days grace)

Assignment:
  Included group: 2ITCSC1A-Studenten (user group)
```

Three effective checks remain after dropping the encryption/boot settings: **OS version, antivirus, firewall** (B40.14 — the rubric asks for "device posture", not encryption).

> **Gotcha: a targeted user without an Intune license shows "Not applicable", not "Non-compliant".** In Verslag40 the policy reported `Total 0 / Not applicable` until the test student got an A5 license (which includes Intune Plan 1). `docent1`/`admin1` were left unlicensed on purpose, which is why Policy 5 is scoped studenten-only (B40.20).
>
> **Gotcha: a stale MDM session blocks evaluation, and a reboot does not fix it.** After licensing, `Device status → Total 0` and `Last contacted` stayed frozen even across a reboot — the MDM session had expired, so the device could not authenticate on the policy-evaluation channel. A fresh user sign-in (not a reboot) restored evaluation; mobile01 then reported **Compliant** (B40.20).
>
> **Gotcha: the Microsoft control plane must bypass SSL-Bump.** Intune device registration and the compliant-device check fail if Squid bumps Microsoft endpoints. Ensure `*.microsoftonline.com`, `enterpriseregistration.windows.net`, `.microsoftazuread-sso.com`, and `.live.com` are on the Squid splice/no-bump list (Runbook 03, B40.9/40.10/44.13).

**Checkpoint Gate 2:**

- [ ] `2ITCSC1A-SASE-Windows-Compliance` created and assigned to `2ITCSC1A-Studenten`
- [ ] Test student has an A5 (Intune) license
- [ ] Intune → Devices → mobile01 (`2ITCSC1A-MOB-1`) shows **Compliant**
- [ ] A NetBird login by that student shows `2ITCSC1A-SASE-Require-Compliant-Device → Report-only: Success` in the CA sign-in tab

---

## [OPTIONAL — NOT DEPLOYED] Step 3: Create NetBird Posture Check (defense-in-depth)

> **Not deployed.** With managed devices, Intune compliance (Step 2) already covers device posture through attestation. The NetBird posture check below is an *optional* defense-in-depth layer — independent timing (tunnel-build) and mechanism (client-side check). It was **not** deployed in the sandbox; the steps are retained as design reference.

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

## [OPTIONAL — NOT DEPLOYED] Step 4: Link posture check to ACL policies

```
NetBird Dashboard → Access Control → Policies
→ Select the relevant policy (Personas-to-Core-Services)
→ Tab: Posture Checks
→ Browse Checks → select "SASE-PoC-Compliance"
→ Add Posture Checks
→ Save Changes
```

Posture checks are per-policy, not global. Link to every policy the persona peers use.

**Checkpoint (optional NetBird posture):**

- [ ] Posture check "SASE-PoC-Compliance" created with all 4 checks
- [ ] Linked to the Personas-to-Core-Services policy
- [ ] mobile01 connects: `netbird status` → Connected
- [ ] Dashboard → Peers → mobile01 → Posture Checks shows "Passed"

---

## Step 5: Validation scenarios

### Scenario 1 — Positive test: compliant device, correct location [VALIDATED]

> **Status:** Validated (Verslag40). 5 CA policies implemented (3 On, 2 Report-only). MFA, legacy-auth blocking, and risk-based blocking are On. Geo-Block and Require-Compliant-Device are Report-only until demo — both already proved "Report-only: Success".

| Step | Action | Expected |
|------|--------|----------|
| 1 | `netbird down` on mobile01 | Tunnel closed |
| 2 | `netbird up` | Microsoft login page opens |
| 3 | Login + MFA | MFA prompt, login succeeds |
| 4 | Wait for tunnel | `netbird status` → Connected |
| 5 | `ping 100.70.154.79` | Response received |
| 6 | Check CA sign-in log | `Resource: Microsoft Graph — Matched`; MFA → Success; Require-Compliant-Device → Report-only: Success; Geo-Block/Legacy-Auth/Risk-Block → Not Applied |

### Scenario 2 — Negative test Gate 1: geo-block

> Geo-Block is Report-only until demo, so the sign-in log records what *would* happen (`Report-only: Failure`); an actual denial occurs only once the policy is flipped On.

| Step | Action | Expected |
|------|--------|----------|
| 1 | Remove Belgium from `2ITCSC1A-SASE-PoC-Allowed-Countries` | Source IP no longer in allowed locations |
| 2 | `netbird down && netbird up` | Login page opens |
| 3 | Login | Report-only: login still succeeds (would be denied when On) |
| 4 | Sign-in log | `2ITCSC1A-SASE-PoC-Geo-Block` → "Report-only: Failure" (→ "Failure"/Block when On) |
| 5 | **Restore:** add Belgium back | Geo-Block returns to "Not applied" |

### Scenario 3 — Negative test Gate 2: OS version (Intune)

| Step | Action | Expected |
|------|--------|----------|
| 1 | Intune → `2ITCSC1A-SASE-Windows-Compliance` → Minimum OS version `10.0.99999.0` | Impossibly high version |
| 2 | Force a sync on mobile01 (Company Portal / `Settings → Accounts → Access work → Sync`) | Device re-evaluated |
| 3 | Intune → Devices → mobile01 | **Non-compliant** |
| 4 | `netbird down && netbird up`; check CA sign-in log | `2ITCSC1A-SASE-Require-Compliant-Device` → "Report-only: Failure" (→ Block when On) |
| 5 | **Restore:** set Minimum OS back to `10.0.22000.0` + sync | Device returns to Compliant |

### Scenario 4 — Negative test Gate 2: antivirus / real-time protection (Intune)

| Step | Action | Expected |
|------|--------|----------|
| 1 | Disable Defender real-time protection: `Set-MpPreference -DisableRealtimeMonitoring $true` (as admin) | RTP off |
| 2 | Force an Intune sync on mobile01 | Device re-evaluated |
| 3 | Intune → Devices → mobile01 | **Non-compliant** (Real-time protection = Require) |
| 4 | `netbird up`; check CA sign-in log | `2ITCSC1A-SASE-Require-Compliant-Device` → "Report-only: Failure" |
| 5 | **Restore:** `Set-MpPreference -DisableRealtimeMonitoring $false` + sync | Compliant again |

> Note: Tamper Protection may need to be disabled first: Settings → Windows Security → Virus & Threat Protection → Tamper Protection: Off. Re-enable immediately after testing.

### Scenario 5 — End-to-end: all three gates

| Step | Gate | Action | Expected |
|------|------|--------|----------|
| 1 | — | `netbird up` | Login page opens |
| 2 | Gate 1 | Login + MFA | Token received; MFA → Success |
| 3 | Gate 2 | Intune compliance attested at login (Policy 5) | Device Compliant → "Report-only: Success" |
| 4 | Gate 3 | Browse `https://google.com` | SSL Bump: cert from SASE-PoC-CA |
| 5 | Gate 3 | Download EICAR test file | ClamAV blocks |
| 6 | Gate 3 | Browse RPZ-blocked domain | DNS NXDOMAIN |
| 7 | — | CA sign-in log | `Resource: Microsoft Graph — Matched`; enforced policies "Success" |
| 8 | — | Intune → Devices → mobile01 | Compliant |

---

## Honest limitations

| Limitation | Why | Status / mitigation |
|-----------|-----|---------------------|
| Two policies in Report-only | Geo-Block and Require-Compliant-Device are deliberately Report-only until demo | Flip to On at demo prep; both already proved "Report-only: Success" (B40.23) |
| Admins persona under no CA policy | `2itcsc1a_admin1` is the only admins member and is the break-glass exclude on every policy (B40.6) | Add a second admin account to govern the admins persona under CA |
| docent1/admin1 not covered by Policy 5 | Left unlicensed on purpose; Policy 5 is scoped studenten-only to avoid a lockout (B40.20) | License docenten/admins for Intune to extend Gate 2 to those personas |
| No Continuous Access Evaluation (CAE) | Gate 2 re-evaluates at authentication and on Intune's periodic cycle, not continuously per-session | Enable CAE for near-real-time revocation |
| GeoIP not 100% accurate | IP geolocation databases have errors | Geo-Block (Gate 1, CA) only; the optional NetBird geo-check (different DB) is not deployed |
| C2 beaconing via WireGuard tunnel | A compromised device can use the WireGuard tunnel (UDP 51820) as a C2 beaconing channel — outside Squid's visibility | Endpoint detection + Suricata (sees WireGuard as encrypted UDP) |
| NetBird posture not deployed | Intune attestation covers device posture un-spoofably; the client-side `process_check` is spoofable (a dummy binary at the path passes) | Deploy NetBird posture (Steps 3–4) only if unmanaged/BYOD devices re-enter scope |

> **Coverage note:** With Intune attestation deployed as Gate 2, the largest gap in the BYOD-era plan — endpoint attestation being process-based and spoofable — is closed: Intune reports actual device state un-spoofably. The main remaining gaps versus commercial SASE (Zscaler ZPA, Netskope Private Access) are Continuous Access Evaluation (CAE) and per-session continuous posture rather than at-authentication evaluation.

See [Concept: Zero Trust](../concepts/zero-trust.md) for the full three-gate model analysis.

---

## Related

- [Component: NetBird](../components/netbird.md)
- [Concept: Zero Trust](../concepts/zero-trust.md)
- [Decision: CA + Posture hybrid (Three-Gate Model)](../decisions/ca-posture-hybrid.md)
- [Decision: Zitadel as IdP broker](../decisions/zitadel-idp-broker.md)

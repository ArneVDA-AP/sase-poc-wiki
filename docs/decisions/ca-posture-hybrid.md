---
title: "Decision: Entra ID CA + NetBird Posture Checks (Hybrid Three-Gate Model)"
tags: [decision, zero-trust, sase, network]
---

# Decision: Entra ID CA + NetBird Posture Checks (Hybrid Three-Gate Model)

**Status:** Implemented — Gate 1 (Entra ID CA, 5 policies) + Gate 2 (Intune device compliance) live since Verslag40 (2 June 2026); Gate 3 (SWG) operational. NetBird posture checks remain an optional, unimplemented defense-in-depth extension.  
**Date:** Architecture April 2026 (Addendum E.2); implemented 2 June 2026 (Verslag40)

## Context

The question this decision answers: how do you verify both *identity* and *device state* before granting network access?

An early position (Verslag18, Bevinding 17.7) — "Entra ID Conditional Access replaces NetBird posture checks" — was first revisited in Verslag27 under the assumption of *unmanaged BYOD*. On unmanaged devices CA cannot check device compliance (that requires Intune enrollment), so NetBird posture checks were proposed as a complementary device-state gate (Doc7, Addendum E.1).

That premise then changed. A scope correction established that the in-scope devices are **managed, Intune-enrolled Windows devices**, not unmanaged BYOD (Addendum E.2; see [Decision: Managed Windows Devices Scope](managed-devices-scope.md)). With managed devices, CA *can* evaluate device state — through Intune compliance attestation. So **Intune compliance became the primary device-posture mechanism (Gate 2)**, and NetBird posture checks were demoted to an **optional defense-in-depth extension** that was never deployed in the sandbox. Verslag40 (2 June 2026) implemented this managed-device model.

## Options considered

> The first three rows are the **BYOD-era analysis** (Addendum E.1 / Doc7). They are retained as decision history; their shared premise — unmanaged BYOD — was superseded by the managed-device scope correction (last row).

| Option | Pro | Con |
|--------|-----|-----|
| **CA only** | Single control point; Microsoft-managed | On *unmanaged* BYOD, CA device compliance is unavailable (needs Intune). CA can enforce MFA and sign-in risk, but not OS version or AV status |
| **NetBird posture only** | Device state without MDM | Evaluates at tunnel-build time only. Cannot evaluate stolen-credential risk, enforce MFA, or detect anomalous sign-in. Process check is spoofable |
| **CA + NetBird posture (BYOD-era plan)** | Each covers what the other cannot | **Premise superseded.** Assumed unmanaged BYOD; once devices were managed, Intune compliance replaced NetBird posture as the device gate |
| **CA + Intune compliance (managed-device — implemented)** | Attestation-based device posture at authentication-time; not spoofable by the end user; same control plane as identity | Limited to Intune-enrolled Windows devices |

## Decision

Managed-device **three-gate model** (Verslag40):

- **Gate 1 — Identity:** Entra ID Conditional Access (5 policies, authentication-time)
- **Gate 2 — Device:** Intune device compliance (attestation-based posture, evaluated at authentication via CA Policy 5 and on Intune's periodic cycle)
- **Gate 3 — Content:** SWG pipeline (Squid SSL-Bump + ClamAV + DLP + Unbound RPZ, every request)

NetBird posture checks remain *available* as an optional, independent defense-in-depth layer (different timing — tunnel-build — and mechanism — client-side check), but were **not deployed**: with managed devices, Intune compliance already covers the rubric's device-posture requirement through attestation rather than a spoofable process check.

**What CA (Gate 1) covers:**
- MFA enforcement
- Sign-in risk evaluation (leaked credentials, impossible travel, anomalous login patterns via Entra ID Protection — available because `aplab.be` has an A5 license which includes P2)
- Named locations (geo-blocking)
- Legacy authentication blocking

**What Intune compliance (Gate 2) covers:**
- OS version (attestation-based, minimum `10.0.22000.0`)
- Defender antivirus active + security intelligence up-to-date + real-time protection
- Windows Defender Firewall active

Because Intune reports actual device state through the management agent (attestation), Gate 2 is **not spoofable by the end user** — the key advantage over the NetBird `process_check` it replaced, which only verifies that a binary exists at a path.

## Implemented Gate 1 policies

| Policy | Status |
|--------|--------|
| CA Policy 1 — MFA required | ✅ Active |
| CA Policy 2 — Geo-block (Belgium only) | Report-only: Success proven — → On at demo |
| CA Policy 3 — Legacy auth blocking | ✅ Active |
| CA Policy 4 — Risk-based blocking | ✅ Active |
| CA Policy 5 — Compliant device required | Report-only: Success proven — → On at demo |

All policies target ALL resources (not a specific app) — CA policies targeting a specific app never fire on NetBird/Zitadel OIDC sign-ins because CA matches on token resource (Microsoft Graph), not client app. This disproves Addendum E section E.2.2's core scoping strategy. User-scoping via persona groups with admin1 as break-glass exclude.

## Implemented Gate 2 controls

| Check | Status |
|-------|--------|
| Intune compliance policy (OS version, Defender AV + firewall, real-time protection) | ✅ Active |
| mobile01 (`2ITCSC1A-MOB-1`) Entra joined + Intune enrolled + compliant | ✅ Verified |

BitLocker/TPM dropped — rubric requires "device posture", not encryption. Three posture checks: OS version, AV, firewall. Policy on Report-only until demo preparation.

## Consequences

- Gate 1 (Entra ID CA) and Gate 2 (Intune compliance) are both live as of Verslag40: four CA policies enforcing (MFA, legacy-auth block, risk-block) plus Policy 5 (compliant device) and Geo-Block held at Report-only until demo. Gate 3 (SWG pipeline) operates independently of identity/device on every request — so all three gates are operational, not just Gate 3
- Activation of Gate 1 requires MFA pre-registration for test accounts before enabling policies (unregistered MFA produces a loop that blocks even the verification session)
- The admins persona falls under no CA policy — `2itcsc1a_admin1` is its sole member and is the break-glass exclude on every policy, so it is deliberately ungoverned to prevent lockout
- Intune Gate 2 depends on the Microsoft control plane being reachable un-bumped: `*.microsoftonline.com` and `enterpriseregistration.windows.net` are on the Squid splice/no-bump list, otherwise device registration and the compliant-device check (Policy 5) break
- **Optional NetBird posture layer (not deployed):** had NetBird posture checks been used as the device gate, the OS check would read the kernel version (`10.0.19041` = Windows 10 2004, the first with the native WireGuard kernel module) and the AV check would be a `process_check` that only verifies a binary exists at a path — spoofable, and blind to whether definitions are current or real-time scanning is active. Intune attestation (Gate 2, minimum OS `10.0.22000.0`) supersedes this; deep content/endpoint protection is Gate 3 (ClamAV)

See also: [Component: NetBird](../components/netbird.md), [Concept: Zero Trust](../concepts/zero-trust.md)

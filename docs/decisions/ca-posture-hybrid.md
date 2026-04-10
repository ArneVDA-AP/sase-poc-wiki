---
title: "Decision: Entra ID CA + NetBird Posture Checks (Hybrid Three-Gate Model)"
tags: [decision, zero-trust, sase, network]
---

# Decision: Entra ID CA + NetBird Posture Checks (Hybrid Three-Gate Model)

**Status:** Architecture validated and documented (Addendum E, April 2026). Gates 1 and 2 not yet activated in sandbox — planned after April 20 evaluation.  
**Date:** April 2026 (Verslag27, Doc7)

## Context

The initial architecture (Verslag18, Bevinding 17.7) stated that "Entra ID Conditional Access replaces NetBird posture checks." This was incorrect. Verslag27 identified and corrected the error after a detailed analysis of what each technology actually does on unmanaged BYOD.

The question: how do you verify both *identity* and *device state* for BYOD students who will not enroll their personal laptops in Intune MDM?

## Options considered

| Option | Pro | Con |
|--------|-----|-----|
| **CA only** | Single control point; Microsoft-managed | CA device compliance checks require Intune enrollment — not available for unmanaged BYOD. CA can enforce MFA and sign-in risk, but not OS version or AV status |
| **Posture checks only** | Device state without MDM | Posture checks evaluate at tunnel-build time — after any CA check would have fired. Cannot evaluate stolen credential risk, enforce MFA, or detect anomalous sign-in |
| **Hybrid (CA + posture)** | Each covers what the other cannot | Two systems to maintain; both marked PLANNED in sandbox |

## Decision

Hybrid model: Entra ID Conditional Access as Gate 1 (authentication-time) + NetBird Posture Checks as Gate 2 (tunnel-build time). Both are complementary, not substitutable.

**What CA (Gate 1) covers that posture cannot:**
- MFA enforcement
- Sign-in risk evaluation (leaked credentials, impossible travel, anomalous login patterns via Entra ID Protection — available because `aplab.be` has A5 license which includes P2)
- Named locations (geo-blocking)
- Legacy authentication blocking

**What posture (Gate 2) covers that CA cannot:**
- OS version verification (no Intune enrollment required)
- AV process verification (`C:\Program Files\Windows Defender\MsMpEng.exe`)
- NetBird client version enforcement
- Geo-check as defense-in-depth (different GeoIP database from CA)

The lesson: always verify what a technology can actually do on the target device type (unmanaged BYOD), not on the ideal device (Intune-enrolled corporate device). CA device compliance checks are structurally unavailable for unmanaged BYOD.

## Planned Gate 1 policies (Addendum E)

All policies targeted at app registration `cebe0d74-be9f-49ac-9f35-65f11586c1bb` (not tenant-wide — `aplab.be` is shared):

| Policy | Mechanism |
|--------|-----------|
| `SASE-PoC-MFA-Required` | Require MFA for all users |
| `SASE-PoC-Geo-Block` | Block access outside named locations (Belgium, Netherlands) |
| `SASE-PoC-Block-Legacy-Auth` | Block legacy authentication protocols |
| `SASE-PoC-Sign-in-Risk` | Block high sign-in risk (Entra ID Protection) |

## Planned Gate 2 checks (Addendum E)

| Check | Threshold |
|-------|-----------|
| `os_version_check` (Windows) | Kernel `10.0.19041` (Windows 10 2004 — first native WireGuard kernel module) |
| `nb_version_check` | Minimum NetBird client version at time of activation |
| `process_check` | `C:\Program Files\Windows Defender\MsMpEng.exe` must be running |
| `geo_location_check` | Belgium (defense-in-depth — different database from Gate 1) |

## Consequences

- Gate 3 (SWG pipeline) is the only gate currently operational in the sandbox — it provides content-level protection regardless of identity/device
- Activation of Gate 1 requires MFA pre-registration for test accounts before enabling policies (unregistered MFA produces a loop that blocks even the verification session)
- OS version check uses kernel version, not marketing version — `10.0.19041` corresponds to Windows 10 2004 (May 2020 Update), the first version with native WireGuard kernel module support
- `process_check` for AV is a baseline compliance check — it does not verify AV definitions are current or that real-time scanning is active; full endpoint protection is Gate 3 (ClamAV)

See also: [Component: NetBird](../components/netbird.md), [Concept: Zero Trust](../concepts/zero-trust.md)

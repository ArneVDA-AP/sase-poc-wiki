---
title: "Decision: Managed Windows Devices Scope (Lector Mandate R11)"
tags: [decision, scope, intune, entra-id, architecture]
---

# Decision: Managed Windows Devices Scope (Lector Mandate R11)

**Status:** Implemented (lector mandate R11, April 21 2026)  
**Date:** April 21, 2026

## Context

The original project scope targeted BYOD (Bring Your Own Device) with an estimated 4000+ personal devices. Under BYOD, device posture enforcement is limited because students will not enroll personal laptops in Intune MDM. Lector mandate R11 changed the scope to managed Windows devices — Intune-managed and Entra ID joined — which fundamentally alters the enforcement model available to the architecture.

## Options considered

| Option | Pro | Con |
|--------|-----|-----|
| **BYOD (original scope)** | Broader device coverage; reflects real-world heterogeneous environment | Device posture checks limited to what NetBird can verify without MDM (OS version, process presence). No compliance attestation. Cannot enforce proxy configuration, firewall rules, or certificates at the OS level |
| **Managed Windows devices (lector mandate)** | Intune compliance attestation available (Gate 2). MDM can push proxy configuration, firewall rules, and certificates. Narrower scope but deeper enforcement | Limited to Windows devices that are Entra joined and Intune enrolled. Does not cover personal devices or non-Windows platforms |

## Decision

Managed Windows devices (Intune-managed, Entra ID joined) as dictated by lector mandate R11. This enables Gate 2 to use Intune compliance policies for device posture verification rather than relying solely on NetBird posture checks. Endpoint enforcement is simplified because MDM can push the proxy PAC file, Windows Firewall rules, and root CA certificates directly to enrolled devices.

## Consequences

- **Gate 2 redesign:** Device posture verification shifts from NetBird posture checks (which verify OS version and process presence at tunnel-build time) to Intune compliance policies (which verify OS version, Defender AV, firewall status, and real-time protection through MDM attestation).
- **Addendum F (Linux PoP transparent proxy)** is largely superseded. The transparent proxy design assumed BYOD devices that could not be configured centrally. With managed devices, proxy configuration is pushed via Intune — transparent interception is no longer the primary deployment model.
- **Identity Bridge mechanism is unchanged.** The JWT-based group synchronization pipeline (Entra ID -> Zitadel -> NetBird -> Identity Bridge -> NATS -> Squid) operates on user identity, not device type. The scope change affects device enforcement, not identity federation.
- **sitepc01 and mobile01** must be Entra joined and Intune enrolled to be valid test endpoints. Both are enrolled (mobile01 in Verslag40, sitepc01/SITE01 in Verslag44), but only **mobile01 (`2ITCSC1A-MOB-1`) has been verified Compliant** (Verslag40). **sitepc01 is not yet verified compliant:** Microsoft Defender is disabled on its Tiny11 image, which fails the AV check of `2ITCSC1A-SASE-Windows-Compliance` — a known compliance gap (Verslag44, B44.12) to be closed by restoring Defender before compliance evaluation is enabled on that endpoint.
- **Certificate deployment:** The internal CA root certificate can be pushed via Intune configuration profile rather than requiring manual installation on each device.

See also: [Decision: Entra ID CA + NetBird Posture Checks](ca-posture-hybrid.md), [Concept: Zero Trust](../concepts/zero-trust.md), [Component: NetBird](../components/netbird.md)

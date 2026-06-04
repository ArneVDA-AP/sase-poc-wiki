---
title: "Concept: Zero Trust"
tags: [zero-trust, sase, network, architecture]
---

# Concept: Zero Trust

**One-line definition:** A security model where no user or device is trusted by default, regardless of network location — access requires continuous verification of identity, device state, and request content at each step.

## How it applies here

Traditional VPN grants network access — once authenticated, users can reach all resources behind the tunnel. Zero Trust replaces this with three orthogonal verification layers, each operating at a different point in the connection lifecycle:

| Gate | Timing | Technology | What it blocks |
|------|--------|-----------|----------------|
| **Gate 1** | Authentication (OIDC login) | Entra ID Conditional Access (5 policies) | Stolen credentials, anomalous sign-in risk, legacy clients, non-allowed geographies |
| **Gate 2** | Authentication + continuous (8h) | Intune device compliance | Outdated OS, no antivirus, no firewall, non-compliant device state |
| **Gate 3** | Every HTTP/DNS request | SWG pipeline (Squid + ClamAV + DLP + Unbound RPZ) | Malware, data exfiltration, blocked domains, known-bad IOCs |

All three gates are operational. Gates 1 and 2 were activated in V40 (CA policies) and V40 (Intune compliance). Some policies are in Report-only mode pending demo preparation (Session 11).

**Why gates are complementary, not redundant:** Gate 1 (CA) enforces MFA and evaluates sign-in risk — but identity alone says nothing about the device's security state. Gate 2 (Intune compliance) attests OS version, antivirus, and firewall — but cannot evaluate stolen-credential risk or enforce MFA. Neither gate can substitute for the other. Gate 3 catches threats in the content that bypass both.

## Where it appears in the stack

- **[NetBird](../components/netbird.md)** — implements Zero Trust Network Access: per-peer identity via OIDC, cryptographic peer keys, granular ACL policies per resource group. A user with access to dc01 cannot reach mgmt01 without a separate policy.
- **[Entra ID CA / Three-Gate Model](../decisions/ca-posture-hybrid.md)** — Gate 1 (identity verification) and Gate 2 (device posture) enforce Zero Trust principles before any tunnel is established.
- **[Squid](../components/squid.md)** + **[ClamAV](../components/clamav-cicap.md)** + **[Python DLP](../components/python-dlp.md)** — Gate 3: assume-breach principle, inspect all traffic regardless of who passed gates 1 and 2.
- **[Suricata](../components/suricata.md)** — lateral movement detection on vtnet1 (LAN), embodying the "assume breach" principle even for internal traffic.
- **[ioc2rpz/Unbound RPZ](../components/ioc2rpz.md)** — DNS-level enforcement: known-malicious domains are blocked before any TCP connection is established.

## Key distinctions

**Zero Trust vs VPN:** VPN gives network access; Zero Trust gives resource access. NetBird implements this via WireGuard mesh + ACL policies — each resource has its own explicit policy, and peers that lack a policy cannot even route to that resource.

**Zero Trust principles (Microsoft framework) — explicit gate mapping:**
- *Verify Explicitly (identity)* → Gate 1: Entra ID CA evaluates who the user is (MFA, sign-in risk, geolocation)
- *Verify Explicitly (device)* → Gate 2: Intune device compliance attests what the device is (OS version, Defender AV, firewall); NetBird posture checks remain optional defense-in-depth
- *Use Least Privilege* → NetBird ACL policies per resource group
- *Assume Breach* → Gate 3: SWG pipeline + Suricata inspect all content in the data plane, regardless of who passed Gates 1 and 2

**Managed devices scope:** Following a scope change (lector mandate, April 21 2026), the project targets managed Windows devices (Intune-managed, Entra joined) rather than BYOD. This enables Gate 2 to use Intune device compliance for attestation-based posture checks (OS version, Defender AV + firewall, real-time protection). NetBird posture checks remain as optional defense-in-depth.

## Gate status

| Gate | Technology | Status |
|------|-----------|--------|
| Gate 1 — Identity | Entra ID Conditional Access (5 policies) | ✅ Operational |
| Gate 2 — Device | Intune device compliance | ✅ Operational (Report-only until demo) |
| Gate 3 — Content | Squid + ClamAV + Python DLP + Suricata + Unbound RPZ | ✅ Operational |

See: [Decision: CA + Posture hybrid](../decisions/ca-posture-hybrid.md)

## Sources

- `raw/Doc7_ZTNA_Context_Aware.md` §1–3 (three-gate model, CA vs posture, implementation plan)
- `raw/SASE_Architectuur_Overzicht.md` §3 (Zero Trust chapter)
- `raw/Doc6_NetBird_ZTNA.md` §1 (Zero Trust network access)

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
| **Gate 1** | Authentication (OIDC login) | Entra ID Conditional Access | Stolen credentials, anomalous sign-in risk, legacy clients, non-allowed geographies |
| **Gate 2** | Tunnel establishment (WireGuard) | NetBird Posture Checks | Outdated OS, no antivirus, non-compliant NetBird client version |
| **Gate 3** | Every HTTP/DNS request | SWG pipeline (Squid + ClamAV + DLP + Unbound RPZ) | Malware, data exfiltration, blocked domains, known-bad IOCs |

Gates 1 and 2 are architecturally designed and documented (Addendum E, April 2026) but not yet activated in the sandbox. Gate 3 is fully operational.

**Why gates are complementary, not redundant:** Gate 1 (CA) can enforce MFA and sign-in risk — but cannot check device state on unmanaged BYOD without Intune enrollment. Gate 2 (posture) can verify OS version and AV status — but cannot evaluate stolen-credential risk or enforce MFA. Neither gate can substitute for the other. Gate 3 catches threats that bypass both.

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
- *Verify Explicitly (device)* → Gate 2: NetBird posture checks evaluate what the device is (OS version, AV, client version)
- *Use Least Privilege* → NetBird ACL policies per resource group
- *Assume Breach* → Gate 3: SWG pipeline + Suricata inspect all content in the data plane, regardless of who passed Gates 1 and 2

**CA device compliance on unmanaged BYOD:** Entra ID CA can check device compliance (OS version, AV status, disk encryption) only for Intune-enrolled devices. BYOD students will not enroll personal laptops in MDM. NetBird posture checks fill this gap — they evaluate device state at WireGuard tunnel-build time without requiring MDM enrollment.

## Sources

- `raw/Doc7_ZTNA_Context_Aware.md` §1–3 (three-gate model, CA vs posture, implementation plan)
- `raw/SASE_Architectuur_Overzicht.md` §3 (Zero Trust chapter)
- `raw/Doc6_NetBird_ZTNA.md` §1 (Zero Trust network access)

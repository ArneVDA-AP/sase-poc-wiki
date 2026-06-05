---
title: "Concept: SASE — Secure Access Service Edge"
tags: [sase, zero-trust, network, architecture]
---

# Concept: SASE — Secure Access Service Edge

**One-line definition:** A framework that converges network transport (WAN Edge) and security services (SSE) into a single cloud-native stack delivered at the edge — where the user is — rather than in a central datacenter.

## How it applies here

This project implements a SASE proof-of-concept for Atlascollege: 4 000 users on managed Windows devices connecting from arbitrary locations, Microsoft 365 as the core app platform. The traditional perimeter model (firewall + VPN = trusted) breaks under these conditions — students are never "inside the walls."

The PoC replaces the perimeter model with a SASE stack where:
- **Transport** is a WireGuard overlay (NetBird) instead of a campus VPN
- **Identity** is Entra ID (Microsoft) instead of network location
- **Inspection** happens at pop01 (the PoP/edge node) on every request, not at a datacenter perimeter

**Control plane / Data plane split** — a deliberate architectural choice:
- **mgmt01** = management plane: distributes configuration (WPAD), manages identity (Zitadel/Entra ID), aggregates threat intelligence (ioc2rpz), runs Docker services
- **pop01** = data plane: inspects all traffic (Squid, ClamAV, Suricata), enforces DNS policy (Unbound RPZ), terminates WireGuard tunnels (wt0)

## Where it appears in the stack

| SASE Pillar | Commercial equivalent | Our implementation | Status |
|-------------|----------------------|-------------------|--------|
| **ZTNA** | Zscaler ZPA | NetBird + Zitadel + Entra ID (three-gate model) | ✅ Operational |
| **SWG** | Zscaler ZIA | Squid + SSL Bump + ClamAV + Python DLP + Unbound RPZ | ✅ Operational |
| **CASB (Layer 1 — Inline)** | Netskope CASB | Squid + Identity Bridge — identity-based URL filtering + DLP pipeline | ✅ Operational |
| **CASB (Layer 2 — API)** | Defender for Cloud Apps | Wazuh + M365 Management Activity API + Graph API Active Response | ✅ Operational |
| **CASB (Layer 3 — Real-time)** | Netskope Real-time | NATS JetStream + Control Daemon — cross-component threat scoring + quarantine | ✅ Operational |
| **FWaaS** | Zscaler Cloud Firewall | OPNsense + Suricata IDS | Partial |
| **SD-WAN** | Zscaler Zero Trust SD-WAN | VyOS site01 + NetBird on sitepc01 | Partial |

## Key distinctions

**SASE vs SSE:** SSE (Security Service Edge) is the security-only subset of SASE — ZTNA + SWG + CASB. SASE adds the WAN Edge component (SD-WAN). This project implements SSE fully and SD-WAN partially.

**PoC vs production:** Commercial SASE uses single-pass inspection engines — TLS decryption, URL filtering, DLP, and malware scanning in one pass. This stack is a serial chain (Squid decrypts → ICAP to Python DLP → ICAP to ClamAV → re-encrypts). Inspection coverage is equivalent; architecture is optimized for clarity over throughput.

**Identity vs network position:** The fundamental shift is that access decisions are based on *who the user is* and *what device they use*, not *where they are on the network*. NetBird ACL policies, Entra ID CA, and the ICAP inspection pipeline collectively enforce this.

## Sources

- `SASE_Architectuur_Overzicht.md` §1–3 (conceptual framing, five pillars, three-gate model)
- `Doc1_Squid_WPAD_PAC.md` §1 (SWG positioning)
- `Doc6_NetBird_ZTNA.md` §1 (ZTNA positioning)

## CASB three-layer model

The CASB implementation spans three enforcement layers, each covering a different time horizon:

- **Layer 1 — Inline (Squid forward proxy):** Identity-based URL filtering and DLP on every HTTP/HTTPS request. Squid enforces two distinct layers in order. First, the UT1 Toulouse category blacklist is identity-blind: it denies a matching domain for *everyone* (student and teacher alike) before any persona rule runs — AI-chat domains such as `chatgpt.com`/`openai.com` sit in this blanket layer and are blocked for both personas. Second, the persona ACL queries the Identity Bridge for the user's persona group and applies differentiated policies on top. Persona differentiation is therefore only observable on a domain that the category blacklist does *not* already block: `deepai.org` is not in UT1, so Studenten are denied (403) while Docenten are allowed (200). Operational since V31.
- **Layer 2 — API-mode (Wazuh + M365 Management Activity API):** Polls SharePoint/OneDrive audit events for policy violations (anonymous shares, external shares). Wazuh custom rules (100600-family) detect violations; Active Response scripts revoke sharing links via Microsoft Graph API. Operational since V39.
- **Layer 3 — Real-time event-driven (NATS + Control Daemon):** All detection silos publish events to the NATS bus. The control daemon maintains per-peer threat scores with sliding-window decay. When a peer exceeds the quarantine threshold, it is removed from policy-bearing persona groups — deny-by-default blocks all connectivity. Operational since V35.

See: [Decision: CASB Three Layers](../decisions/casb-three-layers.md)

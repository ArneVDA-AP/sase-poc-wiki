---
title: "Concept: Behavioral Analysis"
tags: [behavioral-analysis, beaconing, c2, zeek, rita, suricata, sase]
---

# Concept: Behavioral Analysis

**One-line definition:** Detecting threats by how traffic *behaves over time* — regular intervals, abnormal durations, automated call-home patterns — rather than by matching the *content* of individual packets against known-bad signatures.

## How it applies here

This stack runs two detection philosophies side by side, and they answer different questions.

**Signature-based** detection asks "have I seen this exact bad thing before?" [Suricata](../components/suricata.md) matches each packet or flow against tens of thousands of rules describing known malware, exploits and C2 indicators. It is fast and precise, but it is blind to anything not yet in a ruleset — a fresh C2 domain, a custom implant, an unknown beacon.

**Behavioral** detection asks "is this host *acting* compromised, whatever the payload?" That is the job of the [Zeek](../components/zeek.md) + [RITA](../components/rita.md) pair on the SASE_POC parallel stack. The reasoning is simple: a human browsing the web is irregular, but malware checking in with its controller is metronomic. You do not need to recognise the malware to notice the rhythm.

The split between the two tools is deliberate: **Zeek sees, RITA decides.** Zeek passively records every flow into structured logs; RITA imports those logs and scores behaviour across the whole capture window. Nothing is matched against a signature in this path — the verdict comes from the *shape* of the communication.

### The three behaviors detected here

- **Beaconing** — periodic, regular call-home traffic. RITA scores each `source → destination` pair on how regular its interval and payload size are, producing a beacon score from 0 to 1. Legitimate periodic services validate the method: NetBird update checks landed at 100% and Ubuntu NTP at 99% on a known-good network, which is exactly what a working detector should flag. A real C2 beacon would appear in the same list, and the analyst investigates whatever they do not recognise.
- **C2-over-DNS** — command-and-control tunnelled inside DNS queries. This matters because [DNS-level RPZ blocking](rpz.md) cannot stop it on its own: the malicious payload rides in the query names to a domain that may not be on any blocklist. Behavioral analysis spots the abnormal query pattern that signatures and domain blocklists miss.
- **Long connections** — sessions held open far longer than legitimate traffic warrants, a classic indicator of an interactive C2 channel.

Because this is behavioral, not signature, it can flag a destination that no feed has ever listed. When it does — for example RITA scoring `grafana.com` as a Critical beacon at score 0.998 in testing — the detection can be pushed into the [RITA→RPZ pipeline](../decisions/rita-rpz-automation.md) so that a behaviorally-discovered domain becomes a DNS block (NXDOMAIN) enforced on clients. Behavioral detection feeds signature-style enforcement.

## Where it appears in the stack

**[Zeek](../components/zeek.md)** — the sensor. Passively parses flows on the core, Site1 and DC segments into `conn/dns/ssl/http/ntp` logs. Produces the raw telemetry; renders no verdict itself.

**[RITA](../components/rita.md)** — the analyser. Imports Zeek logs into ClickHouse and scores them for beaconing, C2-over-DNS, long connections and threat-intel matches. This is where the behavioral verdict is made.

**[Suricata](../components/suricata.md)** — the contrast. The stack's signature-based IDS on pop01. Behavioral analysis is additive to Suricata, not a replacement: Suricata catches the known, behavioral analysis catches the *abnormal*. Where the sandbox's Suricata and the parallel-stack Zeek/RITA cover the same ground, the sandbox is the source of truth.

**[ioc2rpz / RPZ](../components/ioc2rpz.md)** — the enforcement outlet. A high-confidence behavioral detection (beacon score ≥ 0.8) can be exported into the RPZ feed, turning a behavioral finding into a DNS-level block. See [Decision: RITA→RPZ automation](../decisions/rita-rpz-automation.md).

## Key distinctions

**Behavioral vs signature** — signature detection matches *content* against a known-bad list and is blind to novelty; behavioral detection matches *patterns over time* and can flag a destination no feed has listed, at the cost of needing an analyst to judge intent. They are complementary, not interchangeable. Suricata is signature; Zeek/RITA is behavioral.

**Detection vs enforcement** — behavioral analysis here only *detects*. Zeek and RITA block nothing. Enforcement happens elsewhere: a detection becomes a block only when it is fed into RPZ, where Unbound returns NXDOMAIN. Keep "what RITA found" separate from "what got blocked".

**Score vs severity** — RITA emits both a numeric beacon score and a categorical severity. The score is the reliable signal for automation; RITA's severity labelling is conservative (a clearly periodic destination was rated only "Medium"). The RITA→RPZ extraction filters on beacon score ≥ 0.8 for this reason.

**Beaconing ≠ malicious by itself** — most beacons on a healthy network are legitimate automation (updates, NTP, telemetry). The pattern is suspicious, not the verdict. This is why a whitelist precedes any automated blocking of RITA output.

## Sources

- `rayan_zeek-rita-documentation.md` — Zeek/RITA implementation, multi-worker cluster, beacon validation results.
- `Verslag07_DNS_ThreatIntel_RITA_RPZ_v2.md` — RITA→RPZ pipeline, beacon-score extraction, `grafana.com` end-to-end block.

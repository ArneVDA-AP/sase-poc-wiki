---
title: "Decision: RITA Beacons as a Third Dynamic RPZ Feed"
tags: [decision, rita, rpz, ioc2rpz, dns, beaconing]
---

# Decision: RITA Beacons as a Third Dynamic RPZ Feed

**Status:** Implemented (PoC-validated on the parallel stack — sandbox integration pending)  
**Date:** May 2026 (Verslag07, Deel II)

> **Scope.** Built and proven end-to-end on the **parallel stack** (`mgmt01` = `192.168.122.20`,
> `pop01` = `192.168.122.11`). The pipeline feeds the same ioc2rpz → BIND → Unbound chain the
> sandbox uses, but as a parallel-stack experiment. The sandbox's own RPZ remains the source of
> truth.

## Context

The DNS threat-intelligence chain (see [Runbook 06](../runbooks/06-dns-threat-intel.md)) blocks
domains that already appear in a published feed: URLhaus and ThreatFox. Both are static IOC
lists. They are excellent at known-bad domains and useless against a C2 domain that is not on
anyone's list yet.

RITA closes that gap from the other side. It scores src→dst pairs for beaconing (regular,
machine-like connection intervals) over Zeek's `conn.log`. A high beacon score is behavioral
evidence of C2, independent of any reputation list. The question was how to turn that behavioral
detection into an actual block instead of a dashboard entry an analyst has to act on manually.

## Options considered

| Option | Pro | Con |
|--------|-----|-----|
| **Manual analyst review** | Human judgement on every hit; no false-positive blocking | Slow; a beacon stays reachable until someone looks; defeats the point of automated detection |
| **RITA → file feed → ioc2rpz (chosen)** | Reuses the existing ioc2rpz aggregation, TSIG transfer and Unbound enforcement; a detected domain becomes NXDOMAIN within minutes; fully unattended | Behavioral false positives (NetBird, NTP) need a whitelist; RITA scoring is dataset-window sensitive |
| **RITA → direct Unbound zone write** | One fewer hop | Bypasses ioc2rpz's feed merging and serial management; would reimplement what ioc2rpz already does; loses a single audit point for all three feeds |

## Decision

Wire RITA in as a **third dynamic source** alongside URLhaus and ThreatFox, using a `file:`
source that ioc2rpz already supports.

A cron job (`/opt/scripts/rita_beacon_feed.sh`, every 15 minutes) runs `rita view --beacon`,
keeps rows with **beacon score ≥ 0.8**, strips a whitelist of known-good domains, and writes the
survivors to `/opt/ioc2rpz/cfg/rita_beacons.txt`. ioc2rpz reads that file as the
`rita_beacons` source, merges it into the `threat-intel.rpz.sase` zone next to the two static
feeds, bumps the zone serial, and the existing TSIG AXFR → BIND → Unbound path turns the domain
into an authoritative NXDOMAIN.

Filtering on **score**, not RITA's **severity**, was deliberate. RITA's severity classification
is conservative: `httpbin.org` with 333 connections and score 0.629 only rated "Medium". The raw
score is the more reliable trigger, and 0.8 was the chosen cut-off.

## Consequences

- The block path is fully automated: Zeek captures → RITA scores → cron extracts → ioc2rpz
  merges → BIND transfers → Unbound enforces. A fake beacon (`fake-c2-beacon-test.example.org`)
  was placed by hand and confirmed to reach NXDOMAIN through the whole chain; a live beacon
  (`grafana.com` from the local monitoring stack) was caught, scored Critical, and blocked the
  same way.
- A whitelist (`rita_whitelist.txt`) is mandatory and is the first control point for false
  positives. Without it the RPZ would block NetBird (100% beacon score) and Ubuntu NTP (99%),
  which are legitimate periodic services. See the
  [RITA → RPZ integration runbook](../runbooks/15-rita-rpz-integration.md).
- RITA scoring depends on the analysis window. A rolling import processes the most recent hour
  chunk, so a domain that only beaconed in an earlier window can score low on a later import. A
  beacon must run long enough inside the current window to be caught reliably.
- Propagation latency is bounded by the SOA refresh, lowered to 300 s so Unbound re-pulls the
  zone within five minutes. Whether that fully removes the need for a manual
  `rm zone + restart` on `pop01` was still being verified end-to-end at the time of the report.
- This adds a behavioral feed to a stack that was otherwise reputation-only, without touching the
  enforcement layer. ioc2rpz, BIND and Unbound do not know or care that the third source is
  behavioral rather than a download.

## Related

- [Component: RITA](../components/rita.md)
- [Component: ioc2rpz + BIND + Unbound](../components/ioc2rpz.md)
- [Concept: Behavioral analysis](../concepts/behavioral-analysis.md)
- [Runbook: RITA → RPZ integration](../runbooks/15-rita-rpz-integration.md)
- [Runbook: DNS Threat Intelligence](../runbooks/06-dns-threat-intel.md)

---
title: "Decision: ioc2rpz vs Unbound Native RPZ"
tags: [decision, ioc2rpz, dns, rpz, network]
---

# Decision: ioc2rpz vs Unbound Native RPZ

**Status:** Implemented  
**Date:** April 2026 (Verslag24)

## Context

DNS threat intelligence requires a source that aggregates feed URLs (URLhaus, ThreatFox) into an RPZ zone and keeps it current. Two approaches: use ioc2rpz as a dedicated feed aggregator, or configure Unbound to directly load zone files from static URLs.

## Options considered

| Option | Pro | Con |
|--------|-----|-----|
| **Unbound native RPZ** | No extra component; Unbound 1.24.2 supports `rpz:` block natively | Unbound only loads zone files from disk or via zone transfer — no built-in HTTP feed fetching; would require a cron job to download and convert feeds |
| **ioc2rpz** | Purpose-built feed aggregator; HTTP pull + RPZ zone compilation; GUI for management; NOTIFY to downstream resolvers | Additional Docker container; ioc2rpz.gui has a JavaScript login bug |

## Decision

ioc2rpz as the RPZ zone source, deployed as a Docker container on mgmt01.

ioc2rpz handles the feed lifecycle: it periodically fetches URLhaus and ThreatFox RPZ feeds over HTTP, merges them into a single zone (`threat-intel.rpz.sase`), and sends DNS NOTIFY to downstream servers when the zone updates. Without ioc2rpz, this would require cron jobs to fetch, convert, and reload zone files — reimplementing what ioc2rpz already does.

The GUI JavaScript bug is a known upstream issue with a simple sed fix applied after container start. See [Finding: ioc2rpz GUI JS bug](../findings/ioc2rpz-gui-js-bug.md).

## Consequences

- ioc2rpz Docker container must be running on mgmt01 for zone updates to occur. Initial zone load persists in BIND/Unbound even if ioc2rpz goes down temporarily.
- ioc2rpz sends NOTIFY to `192.168.122.13:53` (Unbound port, not BIND port `53530`). BIND only discovers updates at the next SOA poll interval (3600 s). Manual trigger: `rndc -p 953 retransfer threat-intel.rpz.sase`. Production fix: `pf rdr` to redirect NOTIFY from `192.168.122.23` on port 53 to port 53530.
- The SSL certificate bundled in the ioc2rpz.gui Docker image expired in July 2022. Replace with a fresh self-signed cert before container start.
- ioc2rpz's GUI is proxied by Caddy at `https://ioc2rpz.sandbox.local`.

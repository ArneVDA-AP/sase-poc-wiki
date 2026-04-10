---
title: "Finding: StevenBlack hosts format incompatible with OPNsense Remote ACL"
tags: [finding, squid, workaround]
---

# Finding: StevenBlack hosts format incompatible with OPNsense Remote ACL

**Component:** [Squid](../components/squid.md)  
**Severity:** Gotcha

## What happened

The handbook (v4 §27.1) suggested StevenBlack unified hosts as a Remote ACL source for URL filtering. When configured, OPNsense Remote ACL did not load the list and Squid showed no blacklist entries.

## Root cause

StevenBlack is a `hosts` file: entries in the format `0.0.0.0 ads.example.com`. OPNsense's Remote ACL feature expects a tar.gz archive containing domain lists in Squid ACL format (one domain per line). The formats are incompatible — OPNsense cannot parse a hosts file as a Squid ACL list.

## Resolution / workaround

Use UT1 Toulouse (`dsi.ut-capitole.fr/blacklists`) instead. UT1 is the de facto standard for Squid-based category filtering and is natively supported by OPNsense. It distributes tar.gz archives in Squid ACL format with categories (gambling, malware, phishing, adult) that map directly to OPNsense's ACL interface.

## Lessons

- OPNsense Remote ACL requires tar.gz archives with domain lists in Squid ACL format
- Hosts file format (`0.0.0.0 hostname`) is not compatible
- UT1 Toulouse is the production-tested choice for this use case

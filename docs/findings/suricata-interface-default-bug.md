---
title: "Finding: Suricata 'interface: default' captures only vtnet0"
tags: [finding, suricata, workaround]
---

# Finding: Suricata "interface: default" captures only vtnet0

**Component:** [Suricata](../components/suricata.md)  
**Severity:** Blocker

## What happened

Suricata was configured to monitor WAN and LAN interfaces. vtnet1 (LAN) generated zero alerts despite dc01 producing traffic (apt update, SSH sessions). Rule test cases that should fire on vtnet1 produced no events.

`procstat -f <PID> | grep bpf` showed only one BPF file descriptor open, not two.

## Root cause

`custom.yaml` contained:
```yaml
pcap:
  - interface: default
    checksum-checks: no
```

`interface: default` resolves to only the primary capture interface (vtnet0), not all interfaces. Despite the GUI having both WAN and LAN selected, the `default` keyword in `custom.yaml` overrides this. vtnet1 had a BPF socket open at the OS level but Suricata was not reading from it.

## Resolution / workaround

Explicit per-interface declarations in `/usr/local/opnsense/service/templates/OPNsense/IDS/custom.yaml`:

```yaml
pcap:
  - interface: vtnet0
    checksum-checks: no
  - interface: vtnet1
    checksum-checks: no
```

Apply:
```bash
configctl template reload OPNsense/IDS
configctl ids restart
```

After fix: `procstat -f <PID> | grep bpf` shows two BPF descriptors and vtnet1 events appear within minutes.

## Lessons

- Never use `interface: default` in Suricata custom.yaml — always list each interface explicitly
- `sockstat` does not show Suricata sockets (it uses BPF, not TCP/UDP) — use `procstat -f <PID> | grep bpf`
- Verify the BPF descriptor count matches the number of configured interfaces

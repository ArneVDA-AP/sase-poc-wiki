---
title: "Bevinding: Suricata 'interface: default' neemt alleen vtnet0 op"
tags: [finding, suricata, workaround]
---

# Bevinding: Suricata "interface: default" neemt alleen vtnet0 op

**Component:** [Suricata](../components/suricata.md)  
**Ernst:** Blokker

## Wat er gebeurde

Suricata was geconfigureerd om WAN- en LAN-interfaces te bewaken. vtnet1 (LAN) genereerde geen alerts ondanks dat dc01 verkeer produceerde (apt update, SSH-sessies). Testcases voor regels die op vtnet1 zouden moeten activeren, leverden geen gebeurtenissen op.

`procstat -f <PID> | grep bpf` toonde slechts één BPF-bestandsdescriptor open, niet twee.

## Oorzaak

`custom.yaml` bevatte:
```yaml
pcap:
  - interface: default
    checksum-checks: no
```

`interface: default` wordt omgezet naar alleen de primaire opname-interface (vtnet0), niet alle interfaces. Ondanks dat de GUI zowel WAN als LAN geselecteerd had, overschrijft het `default`-sleutelwoord in `custom.yaml` dit. vtnet1 had een BPF-socket open op OS-niveau maar Suricata las er niet van.

## Oplossing / workaround

Expliciete per-interface-declaraties in `/usr/local/opnsense/service/templates/OPNsense/IDS/custom.yaml`:

```yaml
pcap:
  - interface: vtnet0
    checksum-checks: no
  - interface: vtnet1
    checksum-checks: no
```

Toepassen:
```bash
configctl template reload OPNsense/IDS
configctl ids restart
```

Na de correctie: `procstat -f <PID> | grep bpf` toont twee BPF-descriptors en vtnet1-gebeurtenissen verschijnen binnen minuten.

## Lessen

- Gebruik nooit `interface: default` in Suricata custom.yaml — vermeld altijd elke interface expliciet
- `sockstat` toont geen Suricata-sockets (het gebruikt BPF, geen TCP/UDP) — gebruik `procstat -f <PID> | grep bpf`
- Verifieer dat het aantal BPF-descriptors overeenkomt met het aantal geconfigureerde interfaces

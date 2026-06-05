---
title: "Bevinding: Wazuh indexer crasht op QEMU kvm64 CPU-model"
tags: [finding, wazuh, gns3, qemu]
---

# Bevinding: Wazuh indexer crasht op QEMU kvm64 CPU-model

**Component:** [Wazuh](../components/wazuh.md)  
**Ernst:** Blokkerend (tijdens deployment)

## Wat er gebeurde

De Wazuh indexer (gebouwd op Amazon Linux 2023) crashte onmiddellijk bij het opstarten met illegal instruction-fouten. De container stopte voordat de initialisatie voltooid was, waardoor de volledige Wazuh-stack niet functioneel was.

## Oorzaak

De Wazuh indexer vereist de glibc x86-64-v2 instructieset (Haswell en later). Dit omvat SSE4.2, POPCNT en gerelateerde instructies. Het standaard QEMU `kvm64` CPU-model maskeert deze instructies: het emuleert een generieke x86-64 baseline-CPU die de vereiste feature flags mist.

Wanneer glibc of OpenSearch (de indexer-engine) probeert deze instructies te gebruiken, genereert de CPU een illegal instruction-exceptie en wordt het proces beëindigd.

## Oplossing / workaround

Stel het QEMU CPU-model in op `host` in de GNS3 node-instellingen voor mgmt01:

```
GNS3 → mgmt01 → Configure → QEMU → CPU model: host
```

Het `host`-model geeft de fysieke host-CPU-features door aan de gast-VM, waardoor alle moderne instructiesets beschikbaar worden. Na het wijzigen van deze instelling en het herstarten van het node startte de Wazuh indexer succesvol.

## Lessen

- Verifieer bij het draaien van moderne Linux-containers of -applicaties in GNS3/QEMU altijd de CPU-featurevereisten; `kvm64` is te conservatief voor veel huidige containerimages
- Amazon Linux 2023 en afgeleiden nemen x86-64-v2 als baseline aan, wat niet gegarandeerd wordt door `kvm64`
- Het symptoom (illegal instruction-crash bij opstart) is kenmerkend: als een container onmiddellijk crasht op een QEMU-guest, controleer dan eerst het CPU-model

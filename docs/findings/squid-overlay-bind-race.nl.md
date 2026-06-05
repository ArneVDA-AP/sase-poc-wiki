---
title: "Bevinding: Squid kan overlay-listener niet binden wanneer NetBird wt0 niet klaar is"
tags: [finding, squid, netbird, startup]
---

# Bevinding: Squid kan overlay-listener niet binden wanneer NetBird wt0 niet klaar is

**Component:** [Squid](../components/squid.md), [NetBird](../components/netbird.md)  
**Ernst:** Valkuil

## Wat er gebeurde

Squid kan zijn overlay-listener op `100.70.154.79:3128` niet binden wanneer het start vóórdat NetBird's `wt0`-interface het overlay-IP heeft toegewezen. `sockstat -4 -l | grep ':3128'` toont dan alleen de LAN-listener `10.0.0.1:3128`; de overlay-listener ontbreekt. Het Squid-proces blijft draaien (de bind-failure is niet-fataal), dus van buitenaf lijkt Squid gezond maar is het onbereikbaar op het overlay: clients zien `curl 000` / `TcpTestSucceeded=False` met een lege access.log.

Drie triggers zijn gedocumenteerd: een pop01-reboot (V37.4), een Squid-herstart na een GUI-config/CA-regen (V31.7), en de V43-failover-simulatie (`ifconfig vtnet0 down/up` op pop01) (V44.6).

## Oorzaak

Race condition in de opstartvolgorde: Squid start voordat NetBird klaar is met de tunnelonderhandeling. De wt0-interface en het bijbehorende overlay-IP zijn nog niet beschikbaar op het moment dat Squid probeert te binden. FreeBSD rc.d-volgorde dwingt geen afhankelijkheid af op een volledig opgezette NetBird-overlay.

## Oplossing / workaround

Operationeel herstel: zodra `wt0` up is, bindt een Squid-herstart alsnog:

```bash
sockstat -4 -l | grep ':3128'   # overlay-listener afwezig?
configctl proxy restart          # NIET 'reload' — dat commando bestaat niet op OPNsense (V37.4 / V44.6)
# na afloop: zowel 10.0.0.1:3128 ALS 100.70.154.79:3128 aanwezig
```

Een permanente fix (het starten van Squid blokkeren totdat `wt0` een toegewezen IP heeft, via een rc.d-afhankelijkheid of een wt0-up-hook/watchdog) is **voorgestelde hardening maar blijft een open punt** (V31/V37/V44 open punt 5). Tot dat geïmplementeerd is: controleer de overlay-listener na elke reboot, failover-test of GUI-regen en herstart Squid als die ontbreekt. De gate zou er zo uitzien:

```bash
# voorgesteld (nog niet geïmplementeerd): wacht tot wt0 een IP heeft voordat Squid start
until ifconfig wt0 2>/dev/null | grep -q 'inet 100\.'; do
  sleep 1
done
```

## Lessen

- Elke service die bindt aan een NetBird overlay-IP moet rekening houden met het feit dat de interface nog niet klaar kan zijn. Dit is een structureel probleem, geen eenmalige race
- Squid herprobeert geen mislukte binds na opstart; een mislukte bind is permanent totdat het proces wordt herstart (`configctl proxy restart`)
- De fout is alleen stilzwijgend op proxy-niveau (Squid blijft draaien en access.log blijft leeg), maar de bind-fout **staat wél** in `cache.log`: `commBind Cannot bind socket FD … to 100.70.154.79:3128: (49) Can't assign requested address`. Diagnosticeer via `sockstat` (overlay-listener afwezig) of die cache.log-regel, niet via access.log

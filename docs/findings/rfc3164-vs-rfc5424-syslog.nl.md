---
title: "Finding: RFC3164 vs RFC5424 syslog (rsyslog als normaliser)"
tags: [finding, workaround, telemetry, loki]
---

# Finding: RFC3164 vs RFC5424 syslog (rsyslog als normaliser)

**Component:** [Telemetry Stack](../components/telemetry-stack.nl.md)  
**Ernst:** Gotcha

## Wat er gebeurde

In de v1-telemetriebuild (poc-1a) stuurden netwerkapparaten (OPNsense en VyOS) syslog naar een Promtail-ontvanger, maar Promtail weigerde de packets bij het parsen. De packets kwamen prima aan; ze konden enkel niet geparset worden. Promtail logde:

```
error parsing syslog stream: expecting a version value in the range 1-999 [col 4]
```

Het gevolg: geen enkele log van een netwerkapparaat kwam in Loki, ook al was de connectiviteit correct.

## Oorzaak

Er bestaan twee syslog-wireformaten. **RFC3164** is het oude BSD-formaat en heeft geen versienummer. **RFC5424** is het moderne formaat en begint elk bericht met een versiecijfer (`1`). Netwerkapparaten sturen vrijwel altijd RFC3164, terwijl moderne log tooling (Promtail incluis) RFC5424 verwacht en naar dat eerste versiecijfer zoekt. Een niet-cijfer aantreffen waar de versie hoort te staan (`col 4`) is exact de fout hierboven. Dit is een formaatmismatch, geen netwerk- of configuratiefout.

## Oplossing / workaround

Plaats **rsyslog** als relay/normaliser tussen de apparaten en Promtail:

```
netwerkapparaten ──UDP 514──► rsyslog (host) ──TCP 1514──► Promtail ──► Loki ──► Grafana
                  RFC3164                       RFC5424
```

rsyslog ontvangt het oude formaat op UDP 514, herformatteert het met een RFC5424-template en forwardt het naar Promtail op TCP 1514. De template die de herschrijving doet:

```
template(name="RFC5424Format" type="string"
  string="<%PRI%>1 %TIMESTAMP:::date-rfc3339% %HOSTNAME% %APP-NAME% %PROCID% %MSGID% - %MSG%\n")
```

Bij het aan elkaar knopen doken drie subvalkuilen op:

- **Doe geen `module(load="omfwd")`.** `omfwd` is een built-in rsyslog-module zonder apart `.so`-bestand; het toch laden zorgt ervoor dat rsyslog weigert te starten.
- **AppArmor blokkeert binden op poort 514.** rsyslog kon de geprivilegieerde poort niet binden tot het profiel werd uitgeschakeld (`aa-disable /etc/apparmor.d/usr.sbin.rsyslogd`).
- **De startvolgorde telt.** rsyslog die forwardt naar `127.0.0.1:1514` krijgt `connection refused` als Promtail nog niet draait. Start eerst Promtail, dan rsyslog.

Aan de apparaatkant werd OPNsense geconfigureerd (System → Settings → Logging → Remote) met `RFC 5424` aangevinkt, en de VyOS rolling release had zijn nieuwere `set system syslog remote ...` syntax nodig (de stable `set system syslog host ...` syntax bestaat niet in rolling).

## Lessen

- Vrijwel elk netwerkapparaat spreekt RFC3164; vrijwel elke moderne log tool verwacht RFC5424. Ga bij syslog die "wel aankomt maar niet parset" eerst uit van een formaatmismatch.
- Een rsyslog relay is de schone fix: het normaliseert het formaat op één plek in plaats van elk apparaat of elke consumer aan te passen.
- De **huidige** mgmt01-build omzeilt dit volledig: die lift mee op de bestaande NATS JetStream bus in plaats van een syslog-ontvanger, dus er is geen Promtail en geen RFC3164/RFC5424-conversie in de pipeline. De les blijft genoteerd omdat de valkuil terugkomt in elke syslog-naar-Loki pipeline.

## Gerelateerd

- [Component: Telemetry Stack](../components/telemetry-stack.nl.md)
- [Beslissing: Grafana boven een custom UI](../decisions/grafana-vs-custom-ui.nl.md)
- [Runbook: Telemetry Stack](../runbooks/16-telemetry.nl.md)

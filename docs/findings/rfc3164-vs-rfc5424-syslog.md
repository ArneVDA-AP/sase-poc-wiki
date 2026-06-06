---
title: "Finding: RFC3164 vs RFC5424 syslog — rsyslog as normaliser"
tags: [finding, workaround, telemetry, loki]
---

# Finding: RFC3164 vs RFC5424 syslog — rsyslog as normaliser

**Component:** [Telemetry Stack](../components/telemetry-stack.md)  
**Severity:** Gotcha

## What happened

In the v1 telemetry build (poc-1a), network devices (OPNsense and VyOS) sent syslog to a Promtail receiver, but Promtail rejected the packets at the parse stage. The packets arrived fine — they simply could not be parsed. Promtail logged:

```
error parsing syslog stream: expecting a version value in the range 1-999 [col 4]
```

The result: no network-device logs reached Loki, even though connectivity was correct.

## Root cause

There are two syslog wire formats. **RFC3164** is the old BSD format and has no version number. **RFC5424** is the modern format and begins each message with a version digit (`1`). Network appliances overwhelmingly emit RFC3164, while modern log tooling — Promtail included — expects RFC5424 and looks for that leading version digit. Finding a non-digit where the version should be (`col 4`) is exactly the error above. This is a format mismatch, not a network or configuration fault.

## Resolution / workaround

Insert **rsyslog** as a relay/normaliser between the devices and Promtail:

```
network devices ──UDP 514──► rsyslog (host) ──TCP 1514──► Promtail ──► Loki ──► Grafana
                  RFC3164                      RFC5424
```

rsyslog receives the old format on UDP 514, reformats it with an RFC5424 template, and forwards it to Promtail on TCP 1514. The template that does the rewrite:

```
template(name="RFC5424Format" type="string"
  string="<%PRI%>1 %TIMESTAMP:::date-rfc3339% %HOSTNAME% %APP-NAME% %PROCID% %MSGID% - %MSG%\n")
```

Three sub-traps surfaced while wiring this up:

- **Do not `module(load="omfwd")`.** `omfwd` is a built-in rsyslog module with no separate `.so` file; trying to load it makes rsyslog refuse to start.
- **AppArmor blocks binding port 514.** rsyslog could not bind the privileged port until the profile was disabled (`aa-disable /etc/apparmor.d/usr.sbin.rsyslogd`).
- **Start order matters.** rsyslog forwarding to `127.0.0.1:1514` gets `connection refused` if Promtail is not up yet — start Promtail first, then rsyslog.

On the device side, OPNsense was configured (System → Settings → Logging → Remote) with `RFC 5424` ticked, and VyOS rolling release needed its newer `set system syslog remote ...` syntax (the stable `set system syslog host ...` syntax does not exist in rolling).

## Lessons

- Almost every network appliance speaks RFC3164; almost every modern log tool expects RFC5424. Assume a format mismatch first when syslog "arrives but won't parse".
- An rsyslog relay is the clean fix — it normalises the format in one place instead of patching every device or every consumer.
- The **current** mgmt01 build sidesteps this entirely: it rides the existing NATS JetStream bus instead of a syslog receiver, so there is no Promtail and no RFC3164/RFC5424 conversion in the pipeline. The lesson is still recorded because the trap recurs in any syslog-to-Loki pipeline.

## Related

- [Component: Telemetry Stack](../components/telemetry-stack.md)
- [Decision: Grafana over a custom UI](../decisions/grafana-vs-custom-ui.md)
- [Runbook: Telemetry Stack](../runbooks/16-telemetry.md)

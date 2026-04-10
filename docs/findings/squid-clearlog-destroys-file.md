---
title: "Finding: Squid WebUI 'Clear log' destroys the log file"
tags: [finding, squid, workaround]
---

# Finding: Squid WebUI "Clear log" destroys the log file

**Component:** [Squid](../components/squid.md)  
**Severity:** Gotcha

## What happened

After clicking "Clear log" (→ "flush this log?") in Services → Squid Web Proxy → Diagnostics → Access Log, the access log disappeared entirely. Subsequent proxy tests showed no log entries, suggesting the proxy was not working — but it was working correctly.

This led to a lengthy troubleshooting session chasing a problem that did not exist.

## Root cause

The WebUI "Clear log" button deletes the log file rather than truncating it. After deletion, Squid continues running but writes to a non-existent file descriptor, producing no log output. The log file is not recreated until Squid restarts.

## Resolution / workaround

To truncate the log without deleting it:
```bash
> /var/log/squid/access.log
```

If the file was already deleted:
```bash
configctl proxy restart
```

This causes Squid to recreate the log file on startup.

## Lessons

- Never use the WebUI "Clear log" button for Squid — always truncate from the shell
- When proxy tests show no log output: first check that the log file exists (`ls -la /var/log/squid/access.log`) before assuming the proxy is broken
- Logging-related problems can mask underlying issues — always verify the log file exists before starting proxy debugging

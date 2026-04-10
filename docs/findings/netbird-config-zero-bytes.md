---
title: "Finding: NetBird config.json becomes 0 bytes after unclean shutdown"
tags: [finding, network, workaround]
---

# Finding: NetBird config.json becomes 0 bytes after unclean shutdown

**Component:** [NetBird](../components/netbird.md)  
**Severity:** Gotcha

## What happened

After GNS3 stopped the pop01 node with a SIGKILL (instead of a proper `shutdown -h now`), NetBird failed to start on next boot with no useful error. Interface `wt0` did not exist. All services depending on the overlay IP (`100.70.154.79`) — Squid pre-auth listener, WPAD — were non-functional.

`ls -la /var/db/netbird/config.json` showed 0 bytes.

## Root cause

FreeBSD's UFS filesystem with soft updates writes journal updates asynchronously. When QEMU is killed (GNS3 "Stop node" = SIGKILL to QEMU process), data in the write buffer that has not yet been flushed to disk is lost. The config file's inode exists (the filename is preserved) but the content was not committed to disk before the kill.

This is a known behavior of UFS soft updates on abrupt termination, not a NetBird bug.

## Resolution / workaround

**Prevention:** Back up config.json at the end of each session:
```bash
cp /var/db/netbird/config.json /var/db/netbird/config.json.bak
```

**Recovery:** If config.json is 0 bytes, restore from backup:
```bash
cp /var/db/netbird/config.json.bak /var/db/netbird/config.json
service netbird restart
```

**Additional mitigation:** Add `fsck_y_enable="YES"` to `/etc/rc.conf` on pop01. FreeBSD then runs fsck automatically on next boot after an unclean shutdown, recovering what it can. This does not recover content lost from write buffers, but does fix filesystem inconsistencies.

**Root prevention:** Always shut down OPNsense via `shutdown -h now` before stopping the GNS3 node.

## Lessons

- GNS3 "Stop node" = QEMU SIGKILL — treat it as a power cut
- FreeBSD UFS soft updates are vulnerable to write-buffer loss on abrupt termination
- Back up critical config files (NetBird config.json, any other application state) at session end
- The symptom (0-byte config file, service won't start) is distinctive — check for it first when NetBird is unresponsive after a GNS3 node stop

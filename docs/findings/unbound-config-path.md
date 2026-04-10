---
title: "Finding: Unbound config must go in /usr/local/etc/unbound.opnsense.d/"
tags: [finding, ioc2rpz, workaround]
---

# Finding: Unbound config must go in /usr/local/etc/unbound.opnsense.d/

**Component:** [ioc2rpz](../components/ioc2rpz.md)  
**Severity:** Blocker

## What happened

An RPZ configuration file was placed at `/var/unbound/etc/rpz.conf`. After an Unbound restart triggered by OPNsense, the RPZ configuration was gone and Unbound started without the RPZ module loaded.

## Root cause

`/var/unbound/` is the Unbound chroot directory. OPNsense regenerates this directory on every Unbound restart — all files placed directly in `/var/unbound/etc/` are deleted and replaced by OPNsense's template-generated configuration.

## Resolution / workaround

Place persistent Unbound configuration in:
```
/usr/local/etc/unbound.opnsense.d/rpz.conf
```

OPNsense includes all `.conf` files from this directory in the generated Unbound configuration. Files here survive restarts.

The full RPZ config (`/usr/local/etc/unbound.opnsense.d/rpz.conf`):

```
server:
    module-config: "respip python validator iterator"
rpz:
    name: "threat-intel.rpz.sase"
    zonefile: "/var/unbound/threat-intel.rpz.sase.zone"
    primary: 127.0.0.1@53530
    allow-notify: 127.0.0.1
    rpz-action-override: nxdomain
    rpz-log: yes
    rpz-log-name: "ioc2rpz-threat-intel"
```

Note: `zonefile` still points into `/var/unbound/` — this is correct. The zonefile path is where Unbound stores the transferred zone, which it regenerates via zone transfer. Only persistent *configuration* files need to live in `/usr/local/etc/unbound.opnsense.d/`.

## Lessons

- Never place persistent Unbound config files in `/var/unbound/etc/` — OPNsense overwrites this directory on restart
- The correct path for OPNsense Unbound customization is `/usr/local/etc/unbound.opnsense.d/*.conf`
- This is the same pattern as Squid's `pre-auth/*.conf` — use the designated include directory, not the chroot/template directory

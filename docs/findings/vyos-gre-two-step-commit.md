---
title: "Finding: VyOS GRE tunnel + mirror needs two separate commits"
tags: [finding, vyos, gre, network, zeek]
---

# Finding: VyOS GRE tunnel + mirror needs two separate commits

**Component:** [VyOS](../components/vyos.md)  
**Severity:** Gotcha

> Configured on the **parallel stack** to mirror the Site1 segment to Zeek (`VyOS-1` =
> `192.168.122.31` → `mgmt01` = `192.168.122.20`). Not yet part of the main sandbox.

## What happened

On `VyOS-1`, the Site1 segment mirror is built from two pieces: a GRE tunnel (`tun0`) pointed at
`mgmt01`, and a mirror rule on `eth1` that copies ingress and egress frames into that tunnel.
Configuring both and running a single `commit` fails. VyOS rejects the commit because the mirror
rule references a tunnel interface that does not exist yet at validation time.

## Root cause

VyOS validates the mirror target interface during `commit`. Configuration changes are staged and
only realised when the commit succeeds, so within one commit the `tun0` interface has not been
created at the moment the mirror rule is checked. The mirror rule points at a target VyOS cannot
find, and the whole commit is refused. It is an ordering constraint inside the commit, not a
syntax error.

## Resolution / workaround

Split it into two commits. Create and commit the tunnel first, then add the mirror rules and
commit again:

```bash
configure

# Step 1: create the tunnel, then commit so tun0 actually exists
set interfaces tunnel tun0 encapsulation 'gre'
set interfaces tunnel tun0 source-address '192.168.122.31'
set interfaces tunnel tun0 remote '192.168.122.20'
commit

# Step 2: now the mirror rules can reference tun0
set interfaces ethernet eth1 mirror ingress 'tun0'
set interfaces ethernet eth1 mirror egress 'tun0'
commit
save
```

After step 1 the tunnel is live, so the mirror rule in step 2 validates against a real interface.

Two parameter-name traps showed up here as well. VyOS wanted `source-address` and `remote`, not
`local-ip` / `remote-ip` — the tunnel syntax varies between VyOS versions, so check
`set interfaces tunnel tun0 ?` if a parameter is rejected. The result verifies with:

```
tun0@NONE: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1476
    link/gre 192.168.122.31 peer 192.168.122.20
```

## Lessons

- VyOS commit validation is order-sensitive: a referenced interface must already exist (committed)
  before a rule can point at it. When a rule depends on an interface, commit the interface first.
- "Create the dependency, commit, then create the dependent" is the general pattern for any VyOS
  config where one statement references another resource.
- Tunnel parameter names are version-dependent. `set ... ?` beats guessing.
- This is the segment that *did* work. The DC segment could not be mirrored at all because FreeBSD
  has no kernel mirror — see [Finding: DC inner-segment mirror limit](dc-segment-mirror-limit.md).

## Related

- [Component: VyOS](../components/vyos.md)
- [Component: Zeek](../components/zeek.md)
- [Finding: DC inner-segment mirror limit](dc-segment-mirror-limit.md)
- [Runbook: Zeek & RITA](../runbooks/14-zeek-rita.md)

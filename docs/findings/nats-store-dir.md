---
title: "Finding: NATS JetStream double-nests jetstream/ directory"
tags: [finding, nats, jetstream, configuration]
---

# Finding: NATS JetStream double-nests jetstream/ directory

**Component:** [NATS JetStream](../components/nats-jetstream.md)  
**Severity:** Gotcha

## What happened

Addendum J specified `store_dir: /data/jetstream`. After deployment, NATS created nested `jetstream/jetstream/` directories under `/data/`. Stream introspection tools failed to locate data at the expected path, and manual inspection revealed the double nesting.

## Root cause

NATS automatically appends `jetstream/` to the configured `store_dir` value. When the configuration already includes `jetstream/` in the path, the result is double nesting: `/data/jetstream/jetstream/`. The addendum had redundantly included the suffix that NATS adds on its own.

## Resolution / workaround

Set the correct value:

```
store_dir: /data
```

NATS will then create `/data/jetstream/` as expected. This was empirically confirmed via the NATS startup log (V32).

## Lessons

- Always verify NATS JetStream storage paths in startup logs after deployment — the documentation can be misleading about automatic path suffixing
- The double-nesting does not cause data loss but breaks tooling assumptions about where stream data lives
- When configuring JetStream, specify the parent directory only; NATS adds the `jetstream/` subdirectory itself

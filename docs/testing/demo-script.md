---
title: "Demo Script (per rubric criterion)"
tags: [testing, sase, demo, swg, ztna, casb, sd-wan]
---

# Demo Script (per rubric criterion)

A standalone, interactive presentation script for the final demo. For every grading-rubric
criterion it lists **what you test**, **why that test proves the criterion**, **what you
screencap**, and **what you say** (voiceover). Three pillars get a live five-minute demo; SD-WAN
and the general assessment run via documentation and the presentation.

This page complements the technical test pages — it is the demo/presentation layer, not the test
results themselves:

- [Acceptance Tests (F1–F15)](acceptance-tests.md) — the technical pass/fail tests with commands and expected output.
- [Attack & Bypass Scenarios](attack-scenarios.md) — the bypass attempts per pillar.
- **This page** — rubric-mapped demo choreography: presenter, timing, voiceover, screencap directions, the device matrix, and the enforce-validation checklist.

## Open the document

[**Open the interactive demo script ↗**](../demos/demo-script-rubric.html){target=_blank}

The document is a self-contained HTML page with its own styling and interactive elements
(per-pillar five-minute timers, copy-to-clipboard command blocks, clickable checklists). It opens
full-width in a new tab so the layout and interactivity are preserved.

## What's inside

- **Where you run what** — a device matrix mapping each test to the client it must run on (MOB-1 / SITE01 / node-only), given the current sandbox setup.
- **Test strategy & consolidated results** — the approach and the per-pillar status.
- **Per pillar** (Secure Web Gateway, CASB, ZTNA, SD-WAN, General assessment) — one criterion card each, with Test / Proves / Screencap / Voiceover and the terminal commands.
- **Enforce validation** — the final round that turns the "decision proven, enforcement is the last step" items green (the one-time toggles per test).
- **Preparation & documentation** — the non-demo work needed to make the enforce tests possible.

!!! note "Language"
    The demo script itself is in Dutch (it is the team's presentation aid). Only this wrapper page
    is bilingual; the linked document is shared across both language versions of the wiki.

## Related

- [Testing: Acceptance Tests](acceptance-tests.md)
- [Testing: Attack & Bypass Scenarios](attack-scenarios.md)
- [Concept: SASE](../concepts/sase.md)

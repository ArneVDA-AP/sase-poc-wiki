---
title: "Finding: ioc2rpz GUI JavaScript login bug"
tags: [finding, ioc2rpz, workaround]
---

# Finding: ioc2rpz GUI JavaScript login bug

**Component:** [ioc2rpz](../components/ioc2rpz.md)  
**Severity:** Gotcha

## What happened

Logging into the ioc2rpz GUI at `https://ioc2rpz.sandbox.local` failed with the browser displaying "Unknown error!!!" — the page would briefly appear to process the login, then reload without authenticating.

## Root cause

The `signIn` function in `/opt/ioc2rpz.gui/www/js/io2auth.js` is missing an `e.preventDefault()` call. When the user submits the login form:
1. The browser natively submits the form (HTTP POST, causes page reload)
2. axios also posts the credentials asynchronously
3. The native submit reloads the page before axios completes, aborting the authentication response

The result: the axios login request is sent but the response is never processed. The user sees a page reload with no session established.

This is an upstream bug in the ioc2rpz.gui container image.

## Resolution / workaround

Apply the fix once after container start:

```bash
docker exec ioc2rpz-ioc2rpz-gui-1 sed -i \
  "s|signIn: function(e){ //|signIn: function(e){ e.preventDefault(); //|" \
  /opt/ioc2rpz.gui/www/js/io2auth.js
```

This must be reapplied after each container rebuild (the image does not include the fix).

## Lessons

- JavaScript form submission with both native submit and axios requires `e.preventDefault()` on the form handler — a common frontend bug
- Apply the fix immediately after container start, before any login attempts
- Document the fix command in the operational runbook — it is a recurring maintenance step after container recreation
- For a permanent fix: either fork the image with the patch baked in, or volume-mount a patched `io2auth.js` file over the container's copy so the fix survives container rebuilds

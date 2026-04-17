---
title: "Finding: Suricata Fires Once per SID per Flow (Squid Connection Pooling)"
tags: [suricata, fwaas, squid, finding]
---

# Finding: Suricata Fires Once per SID per Flow (Squid Connection Pooling)

**Component:** [Suricata](../components/suricata.md)  
**Severity:** Insight — not a bug

## What happened

During F9 validation testing, multiple `curl.exe` requests to the same destination via the Squid proxy generated only one Suricata alert per SID, regardless of how many requests were sent. This initially appeared to be alert suppression or a Suricata configuration issue.

```powershell
# mobile01 — repeated three times
curl.exe -x http://100.70.154.79:3128 http://testmyids.com/
curl.exe -x http://100.70.154.79:3128 http://testmyids.com/
curl.exe -x http://100.70.154.79:3128 http://testmyids.com/
# Result: only one SID 2100498 alert in eve.json
```

## Root cause

Squid connection pooling reuses upstream TCP connections. From Suricata's perspective on vtnet0, multiple sequential HTTP requests to the same server appear as traffic within a single TCP flow. Suricata correctly fires once per SID per flow — this is expected behavior by design, not suppression.

The alert count in `eve.json` reflects the number of distinct upstream TCP flows, not the number of client HTTP requests.

## Resolution

No fix needed — this is correct behavior. For testing that requires a new Suricata alert for the same SID:

- Use a different destination host per test run
- Wait for the upstream Squid idle timeout to expire, closing the pooled connection
- Test against distinct URLs that each independently trigger the SID

## Lessons

- When interpreting Suricata alert counts during proxy-based testing, account for Squid connection pooling. One alert per SID per flow is expected and correct.
- This does not affect production detection: genuinely distinct malicious flows will each generate their own alert.
- Investigated and confirmed in Verslag23 (Bevinding 23.6).

## Related

- [Suricata](../components/suricata.md)
- [Squid](../components/squid.md)
- [Testing: Acceptance Tests](../testing/acceptance-tests.md)

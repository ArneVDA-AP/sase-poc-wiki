---
title: "Finding: pyicap Python 3.10+ collections.Callable removed"
tags: [finding, python, dlp, workaround]
---

# Finding: pyicap Python 3.10+ collections.Callable removed

**Component:** [Python DLP](../components/python-dlp.md)  
**Severity:** Blocker

## What happened

The Python DLP ICAP server container crashed on every incoming ICAP request with:
```
AttributeError: module 'collections' has no attribute 'Callable'
```

Squid logged ICAP connection failures and DLP scanning was non-functional.

## Root cause

The `pyicap` library on PyPI uses `collections.Callable` — an attribute that was deprecated in Python 3.3 and removed in Python 3.10. The Docker image uses Python 3.11, so the import fails at runtime on the first ICAP request.

The bug is in the installed `pyicap.py` site-package, not in the project code. The PyPI package has not been updated to use `collections.abc.Callable`.

## Resolution / workaround

Patch pyicap during the Docker build with sed:

```dockerfile
RUN pip install --no-cache-dir pyicap python-docx openpyxl pypdf && \
    sed -i 's/import collections/import collections; import collections.abc/' \
        /usr/local/lib/python3.11/site-packages/pyicap.py && \
    sed -i 's/collections\.Callable/collections.abc.Callable/g' \
        /usr/local/lib/python3.11/site-packages/pyicap.py
```

The two sed commands: (1) add `import collections.abc` alongside the existing `import collections`, (2) replace all uses of `collections.Callable` with `collections.abc.Callable`.

This patch is part of the Dockerfile and applies automatically on every build — no manual intervention required after a container rebuild.

## Lessons

- Third-party libraries may not be maintained to track breaking Python API changes
- Patching in the Dockerfile is a valid approach for upstream bugs in pip packages, provided the patch is clearly documented
- `collections.Callable` → `collections.abc.Callable` is a common Python 3.10+ migration issue; check any library using deprecated `collections` attributes before using it with Python 3.10+

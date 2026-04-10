---
title: "Bevinding: pyicap Python 3.10+ collections.Callable verwijderd"
tags: [finding, python, dlp, workaround]
---

# Bevinding: pyicap Python 3.10+ collections.Callable verwijderd

**Component:** [Python DLP](../components/python-dlp.md)  
**Ernst:** Blokker

## Wat er gebeurde

De Python DLP ICAP-servercontainer crashte bij elk inkomend ICAP-verzoek met:
```
AttributeError: module 'collections' has no attribute 'Callable'
```

Squid logde ICAP-verbindingsfouten en DLP-scanning was niet functioneel.

## Oorzaak

De `pyicap`-bibliotheek op PyPI gebruikt `collections.Callable` — een attribuut dat was verouderd in Python 3.3 en verwijderd in Python 3.10. Het Docker-image gebruikt Python 3.11, waardoor de import mislukt bij het eerste ICAP-verzoek.

De bug bevindt zich in het geïnstalleerde `pyicap.py` site-package, niet in de projectcode. Het PyPI-pakket is niet bijgewerkt om `collections.abc.Callable` te gebruiken.

## Oplossing / workaround

pyicap patchen tijdens de Docker-build met sed:

```dockerfile
RUN pip install --no-cache-dir pyicap python-docx openpyxl pypdf && \
    sed -i 's/import collections/import collections; import collections.abc/' \
        /usr/local/lib/python3.11/site-packages/pyicap.py && \
    sed -i 's/collections\.Callable/collections.abc.Callable/g' \
        /usr/local/lib/python3.11/site-packages/pyicap.py
```

De twee sed-opdrachten: (1) `import collections.abc` toevoegen naast het bestaande `import collections`, (2) alle gebruiken van `collections.Callable` vervangen door `collections.abc.Callable`.

Deze patch maakt deel uit van de Dockerfile en wordt automatisch toegepast bij elke build — geen handmatige tussenkomst vereist na containeropbouw.

## Lessen

- Externe bibliotheken worden mogelijk niet bijgehouden om brekende Python API-wijzigingen bij te houden
- Patchen in de Dockerfile is een geldige aanpak voor upstream-bugs in pip-pakketten, mits de patch duidelijk gedocumenteerd is
- `collections.Callable` → `collections.abc.Callable` is een veelvoorkomend Python 3.10+-migratieprobleem; controleer elke bibliotheek die verouderde `collections`-attributen gebruikt vóór gebruik met Python 3.10+

---
title: "Concept: SSL Bump — HTTPS-interceptie"
tags: [ssl-bump, tls, squid, icap, proxy, sase]
---

# Concept: SSL Bump — HTTPS-interceptie

**Definitie:** Het mechanisme van Squid om HTTPS-verkeer te ontsleutelen door op te treden als TLS man-in-the-middle — het herondertekent servercertificaten on the fly met behulp van een lokale CA die door clients wordt vertrouwd, waardoor inhoudsinspectie van anders ondoorzichtig versleuteld verkeer mogelijk is.

## Hoe het hier van toepassing is

Zonder SSL Bump zien ClamAV en de Python DLP-server alleen een ondoorzichtige versleutelde stroom — nutteloos voor malwarescanning of DLP. SSL Bump maakt HTTPS-inspectie mogelijk: Squid ontsleutelt elke HTTPS-verbinding, geeft de leesbare body door aan de ICAP-inspectieketenm en herencrypteert dan richting de originele server.

De SSL Bump CA voor dit project is `SASE-PoC-CA`, een zelfondertekende root CA geïnstalleerd in de Windows-vertrouwensopslag op mobile01. Zonder dit certificaat verschijnt elke HTTPS-site als een niet-vertrouwde site in de browser.

**TLS-stroom met SSL Bump:**

```
mobile01 (browser) <-> Squid pop01 <-> Internet (originele server)
  [SASE-PoC-CA cert]      ^ leesbare tekst          ^ echte TLS
                     ICAP-inspectie
                 (ClamAV + Python DLP)
```

Squid genereert een per-site-certificaat ondertekend door `SASE-PoC-CA`, en vervangt daarmee het echte certificaat van de originele server. De browser ziet een geldig certificaat (vertrouwd via de geïnstalleerde CA) en detecteert de interceptie niet.

## Waar het in de stack voorkomt

**[Squid](../components/squid.md)** — voert SSL Bump uit op beide listeners. De LAN-listener krijgt ssl-bump-parameters via de OPNsense GUI. De NetBird-overlay-listener (`100.70.154.79:3128`) vereist ssl-bump-parameters in het pre-auth include-bestand — zonder deze tunnelt verkeer door als CONNECT zonder inspectie:

```bash
echo 'http_port 100.70.154.79:3128 ssl-bump cert=/var/squid/ssl/ca.pem \
  dynamic_cert_mem_cache_size=10MB generate-host-certificates=on' \
  > /usr/local/etc/squid/pre-auth/netbird-listener.conf
```

**No-bump-lijst** — sites uitgesloten van SSL-inspectie:
- `login.microsoftonline.com` — **verplicht**: als Squid de Microsoft-aanmeldingspagina bumpt, wordt de Entra ID OIDC-stroom die door NetBird wordt gebruikt gebroken. Alle clientauthenticatie zou mislukken.
- `.microsoft.com`, `.paypal.com`, `.apple.com` — demonstratieve uitzonderingen

**[ClamAV/c-icap](../components/clamav-cicap.md)** en **[Python DLP](../components/python-dlp.md)** — afhankelijk van SSL Bump om leesbare bodies te ontvangen. Zonder SSL Bump kunnen ze geen HTTPS-inhoud inspecteren.

## Belangrijke onderscheidingen

**ssl-bump op pre-auth-listener** — de kritieke bevinding gedocumenteerd in [Bevinding: pre-auth ssl-bump parameters](../findings/pre-auth-ssl-bump-params.md): als het pre-auth include alleen `http_port 100.70.154.79:3128` heeft (geen ssl-bump-parameters), tunnelt HTTPS-verkeer van NetBird-clients als CONNECT zonder inspectie. De volledige ssl-bump-directive moet in de pre-auth conf staan.

**`--ssl-no-revoke` voor Windows curl.exe-tests** — curl.exe op Windows gebruikt de schannel TLS-stack die CRL/OCSP-verificatie probeert op het SSL Bump-certificaat. Zelfondertekende CA's hebben geen revocatie-eindpunt, dus dit mislukt. Voeg `--ssl-no-revoke` toe aan alle Windows CLI-testopdrachten die via de proxy gaan.

**SSL Bump is niet van toepassing op no-bump-sites** — verkeer naar `login.microsoftonline.com` passeert als een CONNECT-tunnel zonder decodering, precies zoals het zou zijn zonder SSL Bump geconfigureerd.

## Bronnen

- `raw/Doc1_Squid_WPAD_PAC.md` §2 (SSL Bump-configuratie, no-bump-lijst)
- `raw/Verslag20.md` (ssl-bump-bevinding voor pre-auth include)
- `raw/Doc2_ClamAV_DLP_Pipeline.md` (afhankelijkheid van SSL Bump voor inspectie)

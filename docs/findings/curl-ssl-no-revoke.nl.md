---
title: "Bevinding: curl --ssl-no-revoke vereist op Windows voor proxytests"
tags: [swg, ssl-bump, finding, workaround]
---

# Bevinding: curl --ssl-no-revoke vereist op Windows voor proxytests

**Component:** [Squid](../components/squid.md), [ClamAV/c-icap](../components/clamav-cicap.md)  
**Ernst:** Valkuil

## Wat er gebeurde

Bij het uitvoeren van ICAP-pijplijn-tests van mobile01 (Windows 11) met `curl.exe` via de Squid-proxy naar een HTTPS-bestemming, mislukt curl met een TLS-fout tenzij `--ssl-no-revoke` wordt opgegeven:

```powershell
# Mislukt zonder --ssl-no-revoke
curl.exe -x http://100.70.154.79:3128 -o eicar_test.txt https://secure.eicar.org/eicar.com

# Werkt wel
curl.exe -x http://100.70.154.79:3128 --ssl-no-revoke -o eicar_test.txt https://secure.eicar.org/eicar.com
```

Zonder `--ssl-no-revoke` bereikt het verzoek de ICAP-pijplijn nooit — curl mislukt bij de TLS-handshake.

## Oorzaak

Windows `curl.exe` gebruikt de Schannel TLS-backend, die CRL- (Certificate Revocation List) en OCSP-validatie probeert voor elk certificaat in de keten. Wanneer Squid SSL Bump uitvoert, wordt het certificaat dat aan de client wordt gepresenteerd uitgegeven door de SASE-PoC-CA — een lokaal gegenereerde CA zonder gepubliceerd CRL- of OCSP-eindpunt. Schannel behandelt een ontbrekend intrekkingseindpunt als een fout en breekt de verbinding af.

## Oplossing

Voeg `--ssl-no-revoke` toe aan alle `curl.exe`-opdrachten van Windows-hosts die via de SSL Bump-proxy naar HTTPS-bestemmingen routeren. Deze vlag instrueert Schannel om intrekkingscontroles over te slaan.

Deze vlag heeft geen invloed op de testgeldigheid — het blokkeer- of doorlaatgedrag van de ICAP-pijplijn is volledig onafhankelijk van de CRL-controle die Schannel uitvoert.

## Lessen

- Alle Windows `curl.exe`-proxytests naar HTTPS-bestemmingen vereisen `--ssl-no-revoke` in deze omgeving.
- Browsertests worden niet beïnvloed: browsers behandelen de SASE-PoC-CA via de geïnstalleerde vertrouwde CA en tolereren ontbrekende CRL-eindpunten soepeler.
- PowerShell `Invoke-WebRequest` gebruikt dezelfde Schannel-backend en heeft hetzelfde gedrag — dezelfde workaround is van toepassing als Invoke-WebRequest wordt gebruikt voor tests.
- In productie zou de SASE-PoC-CA worden gedistribueerd via Groepsbeleid en zou een CRL-eindpunt worden geconfigureerd — waardoor dit probleem niet meer speelt.

## Gerelateerd

- [SSL Bump](../concepts/ssl-bump.md)
- [ClamAV/c-icap](../components/clamav-cicap.md)
- [Squid](../components/squid.md)
- [Testen: Acceptatietests](../testing/acceptance-tests.md)

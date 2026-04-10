---
title: "Squid — Expliciete proxy, WPAD/PAC, SSL Bump, URL-filtering"
tags: [squid, opnsense, proxy, wpad, pac, ssl-bump, tls, icap, sase, caddy, netbird]
---

# Squid — Expliciete proxy, WPAD/PAC, SSL Bump, URL-filtering

**Rol:** Het SWG-ingangspoint — elke HTTP/HTTPS-transactie van een BYOD-client passeert Squid voordat die het internet bereikt. Squid voert URL-filtering en HTTPS-decryptie (SSL Bump) uit en orkestreert de ICAP-inspectiepijplijn.  
**Versie:** Squid 6.x (meegeleverd met OPNsense 25.1)  
**Configuratielocatie:** `/usr/local/etc/squid/squid.conf` (door OPNsense gegenereerd) + `/usr/local/etc/squid/pre-auth/*.conf` (persistente aangepaste directives)

---

## Werking in deze stack

Squid draait in **expliciete proxymodus**: clients worden via een PAC-bestand geconfigureerd om hun HTTP/HTTPS-verkeer rechtstreeks naar Squid te sturen op `100.70.154.79:3128` (de NetBird-overlay-IP van pop01). Dit is het tegenovergestelde van een transparante proxy, waarbij de firewall verkeer stilzwijgend onderschept.

De keuze voor expliciete modus is niet optioneel — transparante proxy via `pf rdr` kan geen via WireGuard gerouteerd verkeer op de `wt0`-interface onderscheppen. Zie [Bevinding: wt0 pf rdr-beperking](../findings/wt0-pf-rdr-limitation.md) en [Beslissing: WPAD/PAC vs. transparante proxy](../decisions/wpad-vs-transparent-proxy.md).

**WPAD/PAC-distributie:** Het PAC-bestand wordt gehost door [Caddy](caddy.md) op mgmt01 via `http://wpad.sandbox.local/wpad.dat`. Clients ontdekken dit via een NetBird Custom DNS Zone die `wpad.sandbox.local` omzet naar de overlay-IP van mgmt01. Mobile01 wordt geconfigureerd via Windows-instellingen → Proxy → "Installatiescript gebruiken".

**SSL Bump:** Squid decrypteert HTTPS-verkeer met een zelfondertekend CA-certificaat (`SASE-PoC-CA`, geïnstalleerd in de vertrouwensopslag van de client). Dit maakt het mogelijk dat de ICAP-pijplijn HTTPS-inhoud inspecteert. Zonder SSL Bump zien ClamAV en de Python DLP-server alleen versleutelde bytes.

**URL-filtering:** Twee blokkeerlijstmechanismen worden uitgevoerd vóór elke `allow`-regel (eerste overeenkomst wint):
1. Handmatige blokkeerlijst: `gambling.com`, `.bet365.com`, `.pokerstars.com`
2. UT1 Toulouse Remote ACL (`dsi.ut-capitole.fr/blacklists`) — categorieën: adult, malware, phishing, gokken

**ICAP-orkestratie:** Squid stuurt verkeer naar twee ICAP-services:
- REQMOD → Python DLP-server op `mgmt01:1345` (POST/PUT/PATCH-uploads)
- RESPMOD → ClamAV/c-icap op `127.0.0.1:1344` (alle responses/downloads)

---

## Configuratie

### Squid-listeners

De OPNsense-GUI genereert één listener op het LAN-IP. Een tweede listener op het NetBird-overlay-IP wordt toegevoegd via het pre-auth include-mechanisme (een gedocumenteerde Squid-functie, geen hack):

```bash
echo 'http_port 100.70.154.79:3128 ssl-bump cert=/var/squid/ssl/ca.pem \
  dynamic_cert_mem_cache_size=10MB generate-host-certificates=on' \
  > /usr/local/etc/squid/pre-auth/netbird-listener.conf
configctl proxy restart
```

De `ssl-bump`-parameters moeten in deze directive worden opgenomen. De GUI voegt ssl-bump alleen toe aan de eigen gegenereerde listener. Zonder deze parameters tunnelt verkeer via het PAC-bestand als gewone CONNECT zonder inspectie. Zie [Bevinding: pre-auth ssl-bump-parameters](../findings/pre-auth-ssl-bump-params.md).

### Toegestane subnetten

`100.64.0.0/10` (NetBird CGNAT-bereik) moet worden toegevoegd aan de Squid-ACL via GUI → Forward Proxy → Access Control List → Allowed Subnets. Zonder dit ontvangen alle NetBird-clients `TCP_DENIED/403`.

### No-bump-lijst

Sites uitgesloten van SSL-inspectie (geconfigureerd via GUI):
- `login.microsoftonline.com` — **verplicht**: beschermt de Entra ID OIDC-flow. Als Squid de Microsoft-inlogpagina bumpt, mislukt NetBird-authenticatie voor alle clients.
- `.microsoft.com`, `.paypal.com`, `.apple.com`, `.banking.example.com` — demonstratie-items.

### ACL-volgorde in squid.conf

```squid
http_access deny blackList
http_access deny remoteblacklist_UT1
http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports
http_access allow localnet
http_access allow subnets
http_access deny all
```

De weigeringsregels voor blokkeerlijsten staan voor alle toestemmingsregels. Squid hanteert eerste-overeenkomst-wint.

---

## Integratiepunten

| Component | Richting | Wat |
|-----------|----------|-----|
| [Caddy](caddy.md) op mgmt01 | → client | Levert PAC-bestand; client gebruikt dit om Squid te vinden |
| [NetBird](netbird.md) | afhankelijkheid | Overlay-IP `100.70.154.79` van wt0 moet bestaan; NetBird DNS-zone levert `wpad.sandbox.local`-resolutie |
| [ClamAV/c-icap](clamav-cicap.md) | ICAP RESPMOD | Squid stuurt alle responses naar c-icap op `127.0.0.1:1344` voor malware- + DLP-scan |
| [Python DLP](python-dlp.md) | ICAP REQMOD | Squid stuurt POST/PUT/PATCH-verzoeken naar mgmt01:1345 voor upload-DLP |
| [Suricata](suricata.md) | parallel | Suricata op vtnet0 ziet de opnieuw versleutelde upstreamverbindingen van Squid; ze delen dezelfde WAN-interface |
| Entra ID / OIDC | no-bump | `login.microsoftonline.com` staat op de no-bump-lijst zodat CA-evaluatie niet verstoord wordt |

---

## Bekende problemen / valkuilen

**"Clear log" in WebUI verwijdert het logbestand** — zie [Bevinding: Squid clearlog](../findings/squid-clearlog-destroys-file.md). Gebruik `> /var/log/squid/access.log` om af te kappen zonder verwijdering.

**Squid-segfault bij stoppen is cosmetisch** — `configctl proxy restart` toont "Segmentation fault" bij het stoppen van Squid. De daemon herstart correct. Geen functionele impact. Gedocumenteerd in meerdere sessies.

**Browsercache maskeert URL-filtering** — gebruik bij het testen van URL-blokkades altijd `curl.exe -x http://100.70.154.79:3128 http://target.com` in plaats van een browser. De browsercache kan een gecachte response leveren ook nadat de URL geblokkeerd is.

**`squid -k parse` segfault aan het einde** — de parse-uitvoer vóór de crash is geldig en nuttig voor diagnose. De segfault is een OPNsense/FreeBSD-eigenaardigheid, geen configuratiefout.

**StevenBlack-hostsformaat is incompatibel** — OPNsense Remote ACL's verwachten een tar.gz-archief in Squid ACL-formaat, geen hostsbestand. Gebruik UT1 Toulouse in plaats daarvan. Zie [Bevinding: StevenBlack incompatibel](../findings/stevenblack-incompatible.md).

---

## Gerelateerd

- [Architectuuroverzicht](../overview/architecture.md)
- [Concept: WPAD/PAC](../concepts/wpad-pac.md)
- [Concept: SSL Bump](../concepts/ssl-bump.md)
- [Concept: ICAP](../concepts/icap.md)
- [Concept: DLP](../concepts/dlp.md)
- [Beslissing: WPAD/PAC vs. transparante proxy](../decisions/wpad-vs-transparent-proxy.md)
- [Beslissing: Twee-laags DLP](../decisions/two-layer-dlp.md)
- [Bevinding: wt0 pf rdr-beperking](../findings/wt0-pf-rdr-limitation.md)
- [Bevinding: pre-auth ssl-bump-parameters](../findings/pre-auth-ssl-bump-params.md)
- [Bevinding: Squid clearlog verwijdert bestand](../findings/squid-clearlog-destroys-file.md)
- [Bevinding: StevenBlack incompatibel](../findings/stevenblack-incompatible.md)

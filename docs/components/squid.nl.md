---
title: "Squid: Expliciete proxy, WPAD/PAC, SSL Bump, URL-filtering"
tags: [squid, opnsense, proxy, wpad, pac, ssl-bump, tls, icap, sase, caddy, netbird]
---

# Squid: Expliciete proxy, WPAD/PAC, SSL Bump, URL-filtering

**Rol:** Het SWG entry point. Elke HTTP/HTTPS-transactie van een overlay-client passeert Squid voordat die het internet bereikt. Squid voert URL-filtering en HTTPS-decryptie (SSL Bump) uit en orkestreert de ICAP inspection pipeline.  
**Versie:** Squid 6.x (meegeleverd met OPNsense 25.1)  
**Configuratielocatie:** `/usr/local/etc/squid/squid.conf` (door OPNsense gegenereerd) + `/usr/local/etc/squid/pre-auth/*.conf` (persistente aangepaste directives)

---

## Werking in deze stack

Squid draait in **expliciete proxymodus**: clients worden via een PAC-bestand geconfigureerd om hun HTTP/HTTPS-verkeer rechtstreeks naar Squid te sturen op `100.70.154.79:3128` (de NetBird-overlay-IP van pop01). Dit is het tegenovergestelde van een transparante proxy, waarbij de firewall verkeer stilzwijgend onderschept.

De keuze voor expliciete modus is niet optioneel: transparante proxy via `pf rdr` kan geen via WireGuard gerouteerd verkeer op de `wt0`-interface onderscheppen. Zie [Bevinding: wt0 pf rdr-beperking](../findings/wt0-pf-rdr-limitation.md) en [Beslissing: WPAD/PAC vs. transparante proxy](../decisions/wpad-vs-transparent-proxy.md).

**WPAD/PAC-distributie:** Het PAC-bestand wordt gehost door [Caddy](caddy.md) op mgmt01 via `http://wpad.sandbox.local/wpad.dat`. Clients ontdekken dit via de `sandbox.local` NetBird Custom DNS Zone, die de query naar `wpad.sandbox.local` naar pop01 Unbound routeert (de NetBird primary nameserver); Unbound resolvet die naar de Caddy-host op het overlay-IP van mgmt01. Mobile01 wordt geconfigureerd via Windows Settings → Proxy → "Use setup script".

**SSL Bump:** Squid decrypteert HTTPS-verkeer met een zelfondertekend CA-certificaat (`SASE-PoC-CA`, geïnstalleerd in de trust store van de client). Dit maakt het mogelijk dat de ICAP pipeline HTTPS-inhoud inspecteert. Zonder SSL Bump zien ClamAV en de Python DLP-server alleen versleutelde bytes.

**URL-filtering:** Twee blokkeerlijstmechanismen worden uitgevoerd vóór elke `allow`-regel (eerste overeenkomst wint):
1. Handmatige blokkeerlijst: `gambling.com`, `.bet365.com`, `.pokerstars.com`
2. UT1 Toulouse Remote ACL (`dsi.ut-capitole.fr/blacklists`): categorieën adult, malware, phishing, gambling

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

De Microsoft control plane-set: deze moeten gesplicet worden, anders breken Intune-apparaatregistratie, Entra-authenticatie en de compliant-device check:
- `login.microsoftonline.com` (**verplicht**): beschermt de Entra ID OIDC-flow. Als Squid de Microsoft-inlogpagina bumpt, mislukt NetBird-authenticatie voor alle clients.
- `.microsoftonline.com`: dekt `device.login.microsoftonline.com` en de overige auth-subdomeinen die de kale FQDN niet dekt.
- `.microsoft.com`: dekt de `manage.microsoft.com`-enrollmentfamilie.
- `enterpriseregistration.windows.net`: het apparaatregistratie-endpoint (exacte host, niet `.windows.net`, dat te breed is).
- `.microsoftazuread-sso.com`: het seamless SSO-endpoint (`autologon.microsoftazuread-sso.com`) is cert-pinned en breekt als het gebumpt wordt.
- `.live.com`: de consumer-auth-leg die tijdens de interactieve Entra-join wordt gebruikt.

Demonstratieve privacy-items: `.paypal.com`, `.apple.com`, `.banking.example.com`.

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
| [Suricata](suricata.md) | parallel | Suricata op vtnet0 ziet de opnieuw versleutelde upstreamverbindingen van Squid, ze delen dezelfde WAN-interface |
| Entra ID / OIDC | no-bump | De Microsoft control plane-set (`.microsoftonline.com`, `.microsoft.com`, `enterpriseregistration.windows.net`, `.microsoftazuread-sso.com`, `.live.com`) staat op de no-bump-lijst zodat apparaatregistratie en CA-evaluatie niet verstoord worden |

---

## NATS-integratie

Squid publiceert proxy access events naar de NATS event bus op mgmt01. Een Python-producerscript op pop01 tailed `access.log` en publiceert gestructureerde events naar `security.alert.proxy`. Events bevatten: bron-IP (overlay), bestemmings-URL, HTTP-methode, responscode, ACL-beslissing en identiteitsgroep (via external_acl). Deze events voeden zowel de [Control Daemon](control-daemon.md) (realtime threat scoring) als [Wazuh](wazuh.md) SIEM (forensische analyse).

De producer filtert op `TCP_DENIED`. Elke HTTPS-block schrijft twee access.log-regels: een `TCP_DENIED/200` op de CONNECT en een `NONE_NONE/403` op de gebumpte GET. Filteren op de `TCP_DENIED`-prefix dedupliceert de block tot één event. Het draagt ook de identiteit: `user=` is enkel gevuld op de `TCP_DENIED` CONNECT-regel (een persona-deny roept de identity-helper aan), terwijl een categorie-deny short-circuit zonder identity-lookup en `user=` leeg laat (`-` wordt `null`). Hetzelfde filter vangt ook plain-HTTP `TCP_DENIED/403`.

Daarnaast integreert Squid met de [Identity Bridge](identity-bridge.md) via `external_acl_type`: voor elk verzoek queryet een helper `http://192.168.122.23:<port>/lookup?ip=%SRC` om het overlay-IP om te zetten naar een Entra ID-personagroep. Dit maakt identiteitsgebaseerde URL-filtering mogelijk (bijv. Studenten geblokkeerd voor deepai.org, Docenten toegestaan). De persona-deny luidt `http_access deny ai_chat persona_studenten`, met de domein-ACL eerst zodat de helper alleen op matchend verkeer wordt geraadpleegd. De include hoort **pre-auth** te staan: de `post-auth/*.conf`-include laadt na `http_access deny all`, waardoor een persona-regel daar dood is. De keten is dan al beëindigd. Een `deny` zo vroeg is veilig omdat die enkel de smalle persona-match vroeg kan weigeren; al het andere valt door naar de security-baseline.

---

## Bekende problemen / valkuilen

**"Clear log" in WebUI verwijdert het logbestand:** zie [Bevinding: Squid clearlog](../findings/squid-clearlog-destroys-file.md). Gebruik `> /var/log/squid/access.log` om af te kappen zonder verwijdering.

**Squid-segfault bij stoppen is cosmetisch.** `configctl proxy restart` toont "Segmentation fault" bij het stoppen van Squid. De daemon herstart correct. Geen functionele impact. Gedocumenteerd in meerdere sessies.

**Browsercache maskeert URL-filtering:** gebruik bij het testen van URL-blokkades altijd `curl.exe -x http://100.70.154.79:3128 http://target.com` in plaats van een browser. De browsercache kan een gecachte response leveren ook nadat de URL geblokkeerd is.

**`squid -k parse` segfault aan het einde.** De parse-uitvoer vóór de crash is geldig en nuttig voor diagnose. De segfault is een OPNsense/FreeBSD-eigenaardigheid, geen configuratiefout.

**StevenBlack-hostsformaat is incompatibel:** OPNsense Remote ACL's verwachten een tar.gz-archief in Squid ACL-formaat, geen hostsbestand. Gebruik UT1 Toulouse in plaats daarvan. Zie [Bevinding: StevenBlack incompatibel](../findings/stevenblack-incompatible.md).

**`--management-url` is verplicht voor zelf-gehoste NetBird:** bij het installeren van de NetBird-agent op pop01 moet de `--management-url`-vlag naar de zelf-gehoste managementserver wijzen. Zonder deze valt de client terug op het NetBird-cloud-management-endpoint en treedt hij nooit toe tot de sandbox-overlay.

**OPNsense-GUI kan Squid niet aan `wt0` binden:** de `wt0`-interface heeft `IPv4 Configuration Type: None` omdat het IP door WireGuard wordt beheerd, niet door de DHCP/statische stack van OPNsense. De Squid-GUI-dropdown toont alleen interfaces met door OPNsense beheerde IP's. Daarom is het pre-auth include-bestand nodig voor de overlay-listener.

**Handboek-NAT-regel (poort 443 → 3129) is irrelevant.** Het handboek beschrijft een NAT-regel voor poort 443 → 3129. Dit is alleen nodig voor transparante proxymodus, waarbij de firewall uitgaand verkeer onderschept. In expliciete proxymodus stuurt de client een `CONNECT`-verzoek rechtstreeks naar Squid op poort 3128. Er is geen DNAT vereist.

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
- [Component: Transparante proxy](transparent-proxy.nl.md) — parallel-stack TPROXY-capture voor verkeer dat de expliciete proxy negeert
- [Identity Bridge](identity-bridge.md)
- [NATS JetStream](nats-jetstream.md)
- [Control Daemon](control-daemon.md)
- [Concept: Identity Flow](../concepts/identity-flow.md)
- [Runbook: Proxy & WPAD](../runbooks/03-proxy-wpad.md)

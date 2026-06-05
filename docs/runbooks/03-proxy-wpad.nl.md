---
title: "Runbook: Proxy & WPAD"
tags: [runbook, squid, ssl-bump, wpad-pac, caddy, swg]
---

# Runbook: Proxy & WPAD

**Bron:** `Doc1_Squid_WPAD_PAC.md`
**Node(s):** pop01 (OPNsense, Squid) + mgmt01 (Caddy, WPAD)
**Vereisten:** [Runbook 02: ZTNA Overlay](02-ztna-overlay.nl.md) afgerond
**Status:** Operationeel

---

## Vereistenchecklist

- [ ] NetBird overlay operationeel; pop01 heeft overlay-IP `100.70.154.79`
- [ ] mgmt01 heeft overlay-IP `100.70.135.241`
- [ ] Caddy draait op mgmt01 (onderdeel van NetBird Docker-stack)
- [ ] OPNsense 25.1 met Squid ingeschakeld op pop01

---

## Stap 1: Transparante proxy uitschakelen

Via OPNsense GUI → Services → Squid Web Proxy → Forward Proxy → General: schakel transparante HTTP-proxy uit. Squid moet alleen hebben:

```
http_port 10.0.0.1:3128
```

De LAN-interface blijft geconfigureerd; Squid vereist minstens één interface met een geldig IP om te starten.

---

## Stap 2: NetBird overlay-subnet toevoegen aan Squid ACL

Zonder dit geeft `curl -x http://100.70.154.79:3128 http://example.com` een `TCP_DENIED/403` terug.

Via GUI: Forward Proxy → Access Control List → Allowed Subnets → voeg `100.64.0.0/10` toe.

Dit genereert:

```squid
acl subnets src 100.64.0.0/10
http_access allow subnets
```

**Verificatie:** `curl -x http://100.70.154.79:3128 http://example.com` toont `TCP_MISS/200` in het toegangslogboek.

---

## Stap 3: Pre-auth listener aanmaken op overlay-IP

De OPNsense GUI kan Squid niet binden aan `wt0` (geen beheerd IP). Gebruik in plaats daarvan het pre-auth include-mechanisme:

```bash
echo 'http_port 100.70.154.79:3128' > /usr/local/etc/squid/pre-auth/netbird-listener.conf
configctl proxy restart
```

> **Valkuil: SSL-Bump-parameters MOETEN later aan deze listener worden toegevoegd (Stap 9).** Een bare `http_port` zonder ssl-bump-vlaggen betekent dat verkeer via deze listener doorkomt als een CONNECT-tunnel zonder inspectie. Dit wordt hersteld in Stap 9; vergeet dit niet.
> Zie [Finding: pre-auth ssl-bump params](../findings/pre-auth-ssl-bump-params.nl.md).

**Verificatie:**

```bash
sockstat -4 -l | grep squid
# Moet TWEE listeners tonen:
# squid  squid  ...  tcp4  10.0.0.1:3128      *:*
# squid  squid  ...  tcp4  100.70.154.79:3128  *:*
```

---

## Stap 4: NetBird-client installeren op mgmt01 + Caddy CA vertrouwen

mgmt01 heeft een NetBird overlay-IP nodig zodat het bereikbaar is voor de WPAD DNS-zone:

```bash
curl -fsSL https://pkgs.netbird.io/install.sh | sudo sh
sudo netbird up --management-url https://netbird.sandbox.local --setup-key 66E94C03-C0E1-4294-B68B-48261ADD6B22
```

De host-level NetBird-daemon vertrouwt de Caddy interne CA niet (Docker-containers doen dit wel via volume-mount). Oplossing:

```bash
docker cp netbird-deploy-caddy-1:/data/caddy/pki/authorities/local/root.crt /tmp/caddy-root.crt
sudo cp /tmp/caddy-root.crt /usr/local/share/ca-certificates/caddy-root.crt
sudo update-ca-certificates
sudo netbird service restart
```

> **Valkuil: Service-herstart is verplicht na CA-import.** Zonder herstart verandert de fout van "context canceled" naar "deadline exceeded"; de daemon draait met de oude trust store.

**Verificatie:** `netbird status` toont Connected, overlay-IP `100.70.135.241`.

---

## Stap 5: Caddy configureren als WPAD-server

Voeg volume-mount toe aan de caddy-service in `docker-compose.yml`:

```yaml
volumes:
  - netbird_caddy_data:/data
  - ./Caddyfile:/etc/caddy/Caddyfile
  - ./wpad:/srv/wpad:ro
```

Voeg twee serverblokken toe aan het Caddyfile (**beide zijn vereist**):

```caddyfile
http://wpad.sandbox.local {
    header /wpad.dat Content-Type application/x-ns-proxy-autoconfig
    root * /srv/wpad
    file_server
}

wpad.sandbox.local {
    tls internal
    header /wpad.dat Content-Type application/x-ns-proxy-autoconfig
    root * /srv/wpad
    file_server
}
```

> **Valkuil: WinHTTP probeert HTTPS zelfs voor HTTP PAC-URL's.** Windows probeert het PAC-bestand eerst via HTTPS op te halen, ongeacht het geconfigureerde URL-schema. Zonder het HTTPS-serverblok toont Caddy `TLS handshake error` in de logs. Het tweede blok met `tls internal` lost dit op.

Maak het PAC-bestand aan op `~/netbird-deploy/wpad/wpad.dat`:

```javascript
function FindProxyForURL(url, host) {
    if (isPlainHostName(host)) return "DIRECT";
    if (shExpMatch(host, "10.*")) return "DIRECT";
    if (shExpMatch(host, "100.64.*")) return "DIRECT";
    if (shExpMatch(host, "127.*")) return "DIRECT";
    if (shExpMatch(host, "*.sandbox.local")) return "DIRECT";
    return "PROXY 100.70.154.79:3128";
}
```

Gebruik `shExpMatch` in plaats van `dnsResolve` of `isInNet`; DNS-gebaseerde PAC-functies kunnen een time-out hebben in de WinHTTP-context.

> **Valkuil: Containerrecreatie vereist, geen herstart.**
> Zie [Finding: Docker volume-recreatie](../findings/docker-volume-recreation.nl.md).

```bash
cd ~/netbird-deploy && docker compose up -d caddy
```

---

## Stap 6: NetBird DNS-zone toevoegen voor WPAD-ontdekking

NetBird Dashboard → DNS → Custom DNS Zones:

```
Zone:          sandbox.local
Distributie:   All
Zoekdomein:    Ingeschakeld
A-record:      wpad → 100.70.135.241 (mgmt01 overlay-IP)
```

**Verificatie:**

```powershell
nslookup wpad.sandbox.local
# Moet resolven naar 100.70.135.241
```

---

## Stap 7: Het ACL-beleid voor proxy-/DNS-bereikbaarheid verifiëren

Onder het V34-personamodel (zie [Component: NetBird](../components/netbird.nl.md)) draagt één allow-beleid alle connectiviteit: **`Personas-to-Core-Services`**: sources `Studenten`/`Docenten`/`Admins` → destination `Core-Services` (pop01 + mgmt01), protocol TCP 3128. Dit laat elke persona-peer (Studenten/Docenten/Admins) de pop01 Squid-proxy bereiken.

> **Verouderd:** eerdere builds gebruikten aparte `Admin-Infrastructure`-, `Mobile-to-Services`- en `Datacenter Access`-policies op `SASE-*`-groepen. Die policies en groepen zijn verwijderd in de V34-migratie; maak ze niet opnieuw aan.

**DNS-bereikbaarheidsvalkuil:** een peer die de pop01-naamserverconfig ontvangt maar er geen ACL-pad naartoe heeft, toont `Nameservers: 0/1 Available` in `netbird status`. Zorg dat de groep van de peer een ACL-pad naar `Core-Services` (pop01 Unbound) heeft.

---

## Stap 8: De client-proxy configureren

### Primaire methode (managed devices): afgedwongen PAC via MDM

Op Intune-enrolled apparaten wordt de PAC-URL vanuit de cloud gepusht in plaats van met de hand gezet, zodat de gebruiker hem niet kan wissen. Een Custom OMA-URI-profiel `2ITCSC1A-SASE-Proxy-PAC` stuurt de NetworkProxy-CSP aan (de Settings Catalog toont NetworkProxy niet, dus loopt het via OMA-URI):

```
./Vendor/MSFT/NetworkProxy/ProxySettingsPerUser   Integer  0
./Vendor/MSFT/NetworkProxy/AutoDetect             Integer  0
./Vendor/MSFT/NetworkProxy/SetupScriptUrl         String   http://wpad.sandbox.local/wpad.dat
```

`ProxySettingsPerUser=0` maakt de PAC systeembreed, `AutoDetect=0` forceert de expliciete PAC in plaats van WPAD-discovery, en `SetupScriptUrl` wijst naar het Caddy PAC-doel uit Stap 5. Zet geen ProxyServer-node naast de PAC-URL; die combinatie is een bekende Intune-bug die het profiel sloopt. Zie [Intune Endpoint Enforcement](../components/intune-endpoint-enforcement.md) voor het volledige profiel en de device-scoping.

### Fallback-methode (handmatig): Windows-proxy op mobile01

Voor een unmanaged of met de hand gebouwde testclient zet je dezelfde PAC-URL handmatig:

```
Windows Instellingen → Netwerk en internet → Proxy:
  Instellingen automatisch detecteren: Uit
  Installatiescript gebruiken: Aan
  Scriptadres: http://wpad.sandbox.local/wpad.dat
  Handmatige proxy: Uit
```

De Caddy root-CA moet al zijn geïmporteerd in de Windows-trust store (gedaan tijdens NetBird-registratie).

---

## Stap 9: SSL Bump inschakelen

**Maak SASE-PoC-CA aan** via System → Trust → Authorities:
- Methode: Create an internal Certificate Authority
- Sleuteltype: RSA, 2048 bit, SHA256
- Geldigheidsduur: 365 dagen
- Organisatie: SASE PoC

**Schakel SSL-inspectie in** in Forward Proxy-instellingen met de nieuwe CA.

**Werk de pre-auth include bij met volledige ssl-bump-parameters:**

```bash
echo 'http_port 100.70.154.79:3128 ssl-bump cert=/var/squid/ssl/ca.pem dynamic_cert_mem_cache_size=10MB generate-host-certificates=on' > /usr/local/etc/squid/pre-auth/netbird-listener.conf
configctl proxy restart
```

**Verificatie:** Browser toont certificaat met issuer `Organization (O): SASE PoC`.

**Configureer de no-bump-lijst** (GUI → Forward Proxy → SSL-inspectie):

```
login.microsoftonline.com
.microsoftonline.com
.microsoft.com
enterpriseregistration.windows.net
.microsoftazuread-sso.com
.live.com
.paypal.com
.apple.com
.banking.example.com
```

> **Valkuil: de Microsoft control plane-set MOET in de no-bump-lijst staan.** Als Squid een van deze bumpt, wordt de OIDC-certificaatketen verbroken en mislukken Entra ID-authenticatie, Intune-apparaatregistratie en de conform-apparaatcontrole voor alle NetBird-clients. `.microsoftonline.com` (met punt vooraan) dekt `device.login.microsoftonline.com`; `enterpriseregistration.windows.net` is de exacte apparaatregistratie-host; `.microsoftazuread-sso.com` is het cert-pinned seamless SSO-endpoint; `.live.com` is de consumer-auth-leg die tijdens de interactieve Entra-join wordt geraakt.

---

## Stap 10: URL-filtering inschakelen met UT1 Toulouse

Via GUI → Forward Proxy → Remote Access Control List:

```
Bestandsnaam:    UT1
URL:             https://dsi.ut-capitole.fr/blacklists/download/blacklists.tar.gz
SSL cert negeren: aangevinkt
Categorieën:     adult, malware, phishing, gambling
```

Handmatige zwarte lijst via Forward Proxy → Access Control List → Blacklist:

```
gambling.com
.bet365.com
.pokerstars.com
```

**Verificatie:** gebruik curl, niet een browser (browsercache maskeert blokkering):

```powershell
curl.exe -x http://100.70.154.79:3128 http://gambling.com -v
# Verwacht: HTTP/1.1 403 Forbidden, X-Squid-Error: ERR_ACCESS_DENIED 0
```

---

## Bekende cosmetische problemen

**OPNsense "Clear log" verwijdert het logbestand:** de WebUI-knop verwijdert in plaats van afkapt. Squid stopt met loggen. Oplossing:

```bash
> /var/log/squid/access.log   # afkappen zonder verwijderen
```

Zie [Finding: Squid clearlog vernietigt bestand](../findings/squid-clearlog-destroys-file.nl.md).

**Squid segfault bij stoppen is cosmetisch:** treedt op tijdens de stopfase van `configctl proxy restart` op FreeBSD. De daemon herstart correct. Geen functionele impact.

---

## Eindverificatie

- [ ] `sockstat -4 -l | grep squid` toont beide listeners (10.0.0.1 en 100.70.154.79)
- [ ] `curl -x http://100.70.154.79:3128 http://example.com` geeft 200 terug
- [ ] `nslookup wpad.sandbox.local` resolveert naar `100.70.135.241`
- [ ] Browser op mobile01 configureert proxy automatisch via PAC
- [ ] HTTPS-sites tonen certificaat-issuer "SASE PoC"
- [ ] `login.microsoftonline.com` toont zijn eigen certificaat (niet gebumpt)
- [ ] `curl.exe -x http://100.70.154.79:3128 http://gambling.com` geeft 403 terug

---

## Gerelateerd

- [Component: Squid](../components/squid.nl.md)
- [Component: Caddy](../components/caddy.nl.md)
- [Concept: SSL Bump](../concepts/ssl-bump.nl.md)
- [Concept: WPAD/PAC](../concepts/wpad-pac.nl.md)
- [Beslissing: WPAD/PAC vs Transparante Proxy](../decisions/wpad-vs-transparent-proxy.nl.md)
- [Finding: wt0 pf rdr beperking](../findings/wt0-pf-rdr-limitation.nl.md)
- [Finding: pre-auth ssl-bump params](../findings/pre-auth-ssl-bump-params.nl.md)
- [Finding: StevenBlack incompatibel](../findings/stevenblack-incompatible.nl.md)
- [Finding: Squid clearlog vernietigt bestand](../findings/squid-clearlog-destroys-file.nl.md)
- [Finding: Docker volume-recreatie](../findings/docker-volume-recreation.nl.md)

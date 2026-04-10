---
title: "Caddy — WPAD-server, Reverse Proxy, TLS-terminator"
tags: [caddy, network, sase, proxy, tls, wpad]
---

# Caddy — WPAD-server, Reverse Proxy, TLS-terminator

**Rol:** Multifunctionele HTTP-server op mgmt01 — levert het WPAD PAC-bestand voor automatische browserconfiguratie, biedt TLS-terminatie voor de NetBird/Zitadel-stack en reverse-proxiet de ioc2rpz-beheer-GUI.  
**Versie:** Caddy 2.x (onderdeel van de NetBird Docker Compose-stack op mgmt01)  
**Configuratielocatie:** `~/Caddyfile` op mgmt01, geleverd door de `caddy`-container in de NetBird Docker-stack

---

## Werking in deze stack

Caddy op mgmt01 vervult drie afzonderlijke rollen die toevallig hetzelfde proces delen:

**1. WPAD/PAC-bestandsserver** — Caddy levert `/wpad.dat` op `http://wpad.sandbox.local` via de overlay-IP `100.70.135.241` van mgmt01. BYOD-clients ontdekken deze URL automatisch wanneer hun DNS-zoekdomein `sandbox.local` is: de browser vraagt `wpad.sandbox.local`, krijgt een antwoord, haalt `/wpad.dat` op en voert het PAC-bestand JavaScript uit om te ontdekken dat al het verkeer naar `PROXY 100.70.154.79:3128` (pop01 Squid) moet. Zie [Concept: WPAD/PAC](../concepts/wpad-pac.md).

**2. NetBird TLS-terminator** — Het quickstart-script van NetBird installeert Caddy als de TLS-terminerende reverse proxy voor alle NetBird- en Zitadel-services op `netbird.sandbox.local`. Het TLS-certificaat is zelfondertekend via de `tls internal`-directive van Caddy (de ingebouwde lokale CA van Caddy). Let's Encrypt kan niet worden gebruikt omdat `netbird.sandbox.local` niet publiek oplosbaar is. Zie [Beslissing: Zitadel IdP-broker](../decisions/zitadel-idp-broker.md).

**3. ioc2rpz GUI-proxy** — Caddy reverse-proxiet de ioc2rpz GUI-container naar `https://ioc2rpz.sandbox.local`. Zie [Component: ioc2rpz](ioc2rpz.md).

Caddy maakt deel uit van de control plane-rol van mgmt01: het distribueert configuratie (WPAD), maakt beheerinterfaces mogelijk (NetBird, ioc2rpz) en raakt het dataplaneverkeer dat via pop01 stroomt nooit aan.

---

## Configuratie

### PAC-bestand (wpad.dat)

Geleverd op `http://wpad.sandbox.local/wpad.dat` met MIME-type `application/x-ns-proxy-autoconfig`:

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

Het PAC-bestand gebruikt `shExpMatch()` in plaats van `isInNet()` of `dnsResolve()` — DNS-gerelateerde PAC-functies kunnen een timeout veroorzaken in de WinHTTP-context. `isPlainHostName()` retourneert DIRECT voor single-label hostnamen (zonder punt). Het overlay-subnet (`100.64.*`), loopback (`127.*`), DC-LAN (`10.*`) en alle `*.sandbox.local` interne namen omzeilen de proxy. Al het andere gaat via Squid op pop01.

### Caddyfile — NetBird TLS

Het Caddyfile wordt gegenereerd door het NetBird-quickstart-script en vervolgens gepatcht om `tls internal` toe te voegen:

```caddyfile
netbird.sandbox.local {
    tls internal

    # reverse_proxy naar Zitadel en NetBird-beheer (door script gegenereerd)
}
```

Een Docker-netwerkalias maakt `netbird.sandbox.local` oplosbaar binnen het Docker-netwerk:

```yaml
services:
  caddy:
    networks:
      netbird:
        aliases:
          - netbird.sandbox.local
```

Deze alias staat Zitadel en NetBird-beheercontainers toe Caddy intern te bereiken voor HTTPS-callbacks.

### Caddyfile — WPAD en ioc2rpz

De WPAD- en ioc2rpz-blokken worden toegevoegd aan hetzelfde Caddyfile:

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

ioc2rpz.sandbox.local {
    tls internal
    reverse_proxy https://192.168.122.23:8444 {
        transport http {
            tls_insecure_skip_verify
        }
    }
}
```

Twee WPAD-blokken zijn vereist: WinHTTP probeert het PAC-bestand eerst via HTTPS op te halen, zelfs wanneer de geconfigureerde URL `http://` gebruikt. Zonder het HTTPS-blok tonen Caddy-logs TLS-handshake-fouten van de client. Het HTTPS-blok gebruikt `tls internal` (de ingebouwde lokale CA van Caddy).

De ioc2rpz GUI-container luistert op HTTPS-poort 8444 met een zelfondertekend certificaat. Caddy proxiet naar `https://192.168.122.23:8444` met `tls_insecure_skip_verify` om het zelfondertekende certificaat te accepteren.

### Wijzigingen toepassen

Volume-mount-wijzigingen vereisen containerrecreatie, niet alleen herstart:
```bash
docker compose up -d caddy   # past nieuwe mounts toe
# docker compose restart caddy  ← past GEEN nieuwe volume-mounts toe
```

---

## Integratiepunten

| Component | Richting | Wat |
|-----------|----------|-----|
| [NetBird](netbird.md) | TLS | Caddy termineert TLS voor `netbird.sandbox.local` met `tls internal` CA |
| [Squid](squid.md) | → browser | Caddy levert WPAD PAC-bestand; clients gebruiken het om het adres van Squid te ontdekken |
| [ioc2rpz](ioc2rpz.md) | reverse proxy | Caddy proxiet de ioc2rpz GUI op `https://ioc2rpz.sandbox.local` |

---

## Bekende problemen / valkuilen

**`tls internal` geeft browsercertificaatwaarschuwing** — de interne CA van Caddy wordt standaard door geen enkele browser vertrouwd. Voor het NetBird Dashboard betekent dit het accepteren van een beveiligingsuitzondering. De NetBird OIDC-flow vereist alleen HTTPS (niet een publiek vertrouwd certificaat), dus het zelfondertekende certificaat is functioneel voldoende.

**Docker-volumerecreatie** — nieuwe volume-mounts in `docker-compose.yml` voor de `caddy`-service zijn pas van kracht na `docker compose up -d caddy`, niet `docker compose restart caddy`. De `restart`-opdracht hergebruikt de bestaande container zonder volume-mount-wijzigingen opnieuw te lezen.

**WPAD vereist zowel HTTP- als HTTPS-blokken** — WinHTTP probeert het PAC-bestand eerst via HTTPS op te halen, zelfs wanneer de geconfigureerde URL `http://` specificeert. Zonder een HTTPS-serverblok voor `wpad.sandbox.local` tonen Caddy-logs `TLS handshake error` van de client. Het Caddyfile moet zowel een `http://wpad.sandbox.local`-blok als een `wpad.sandbox.local`-blok met `tls internal` bevatten.

---

## Gerelateerd

- [Architectuuroverzicht](../overview/architecture.md)
- [Concept: WPAD/PAC](../concepts/wpad-pac.md)
- [Component: Squid](squid.md)
- [Component: NetBird](netbird.md)
- [Component: ioc2rpz](ioc2rpz.md)
- [Beslissing: WPAD/PAC vs. transparante proxy](../decisions/wpad-vs-transparent-proxy.md)
- [Beslissing: Zitadel als IdP-broker](../decisions/zitadel-idp-broker.md)

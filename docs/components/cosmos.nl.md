---
title: "Cosmos: Identity-Aware Application Gateway"
tags: [cosmos, application-gateway, reverse-proxy, mfa, smartshield, zero-trust, ztna, sase, docker, dns]
---

# Cosmos: Identity-Aware Application Gateway

**Rol:** Identity-aware application gateway op dc01, een reverse proxy die voor elke interne DC-service een per-applicatie login + MFA-gate, SmartShield rate-limiting en Docker-container-isolatie zet.
**Versie:** Cosmos Server v0.22.10
**Configuratielocatie:** dc01 (`10.0.0.100`), `/var/lib/cosmos` (gemount als `/config`); Cosmos admin-UI op `https://cosmos.dc.local/cosmos-ui/`
**Status:** ✅ PoC-gevalideerd op de parallelle stack, sandbox-integratie pending

> **Sidenote, parallelle-stack-scope.** Cosmos is gebouwd en gevalideerd op de parallelle `SASE_POC`-omgeving (dc01 `10.0.0.100`), nog niet in het sandbox-geheel geïntegreerd. Waar de scope overlapt met de sandbox (DNS, de NetBird-overlay) is de sandbox de bron van waarheid en is Cosmos additief: het voegt er een application-admission-laag bovenop toe, het vervangt niets in de sandbox-build.

---

## Werking in deze stack

De stack heeft al drie lagen onder Cosmos: NetBird ZTNA bepaalt welk device het DC-netwerk mag betreden, OPNsense + Suricata doen L3/L4-filtering en IDS, en Squid + ClamAV inspecteren uitgaande content. Geen van die lagen beantwoordt de vraag *welke gebruiker deze specifieke applicatie mag openen*. Cosmos is de vierde laag die dat wel doet.

Concreet is elke DC-service (Uptime Kuma, Gitea, toekomstige services) **uitsluitend** bereikbaar via Cosmos op poort 80/443. De service-containers zijn nooit extern gebonden: Kuma's `3001/tcp` en Gitea's `3000/tcp` staan niet op de host blootgesteld, dus het enige pad naar binnen is de reverse proxy. Cosmos onderschept elk verzoek naar een beveiligde route, draait login + MFA, en forwardt pas daarna naar de backend-container. Dit is de tweede van twee Zero Trust-gates: NetBird is network admission (device-niveau), Cosmos is application admission (user-niveau). Een device dat op de NetBird-overlay zit maar waarvan de Cosmos-sessie verlopen is, kan geen enkele DC-applicatie openen zonder opnieuw met MFA te authenticeren. Zie [Beslissing: Twee-laags ZTNA](../decisions/cosmos-two-layer-ztna.nl.md).

Cosmos bundelt vijf afzonderlijke functies in één component:

1. **Reverse proxy met container-isolatie.** Elke service draait in zijn eigen Docker-netwerk (`cosmos-Uptime-Kuma-default`, `cosmos-Gitea-default`). Een aanvaller die één container compromitteert kan niet lateraal naar de andere bewegen: de netwerken zijn geïsoleerd en de backend-poorten zijn niet extern gebonden.
2. **Identity gate met MFA.** Cosmos verifieert de gebruikersidentiteit via login + TOTP (Microsoft / Google Authenticator) voor het toegang verleent. De sessie is tijdsgebonden: 48 uur in deze PoC. Een verlopen sessie, een andere browser of een incognito-venster forceert opnieuw authenticeren met MFA.
3. **SmartShield (per route, v0.22.10).** Rate-limiting, bot-detectie, IP-gebaseerde throttling en anti-DDoS per route. Uptime Kuma staat op een drempel van 12.000 requests/minuut. Omdat elke client al door NetBird moet voor hij Cosmos bereikt, ziet SmartShield NetBird overlay-adressen als source IP: internet-scanners bereiken Cosmos überhaupt nooit.
4. **Container-management en zichtbaarheid.** Het Servapps-paneel geeft de admin zicht op elke Docker-container op dc01, inclusief containers die niet via Cosmos zijn gedeployed. Dit is de basis voor inbound Shadow IT-zichtbaarheid op dc01 (zie [Beslissing: Twee-laags ZTNA](../decisions/cosmos-two-layer-ztna.nl.md)).
5. **Session management als access control.** NetBird geeft device-level netwerktoegang; Cosmos voegt daar user-level, tijdsgebonden toegang bovenop. Het verlopen van een sessie is op zich een access-control-event: het device blijft op het netwerk, maar de applicatie sluit tot de volgende MFA.

### Uptime Kuma achter de gate

Uptime Kuma is het monitoring-dashboard van de stack, gedeployed via Cosmos **met** de MFA-gate aan, bereikbaar op `https://kuma.dc.local`. Het dashboard toont operationeel gevoelige staat (welke services up zijn, historische uptime over de SASE-stack) en krijgt daarom de maximale stack: NetBird (ZTNA) + Cosmos MFA + Kuma's eigen login. Het monitort de parallelle-stack-componenten op IP (OPNsense `10.0.0.1`, Cosmos `10.0.0.100`, mgmt01 `192.168.122.20`, Gitea via het Cosmos-IP). De demo-flow is: incognito-venster → `https://kuma.dc.local` → Cosmos-login (niet Kuma) → credentials → MFA-code → dashboard.

Gitea is via Cosmos gedeployed **zonder** de MFA-gate, een bewuste keuze per service: Gitea heeft zijn eigen volwassen auth (SSH keys, PAT's, optionele 2FA), en Cosmos MFA daarbovenop zou dubbele authenticatie zijn voor dezelfde identiteit. Zie [Beslissing: Twee-laags ZTNA](../decisions/cosmos-two-layer-ztna.nl.md) voor de motivatie per service.

---

## Configuratie

Belangrijke keuzes en de reden per keuze:

| Setting | Waarde | Reden |
|---------|--------|-------|
| Hostname | `10.0.0.100` | Gesloten lab, geen FQDN-migratie gedaan; dit beperkt OAuth/SSO cross-domain. Zie valkuilen + [Bevinding: hostname op IP beperkt OAuth](../findings/cosmos-hostname-oauth.nl.md). |
| HTTPS Mode | `SELFSIGNED` | Geen Let's Encrypt bereikbaar in een gesloten lab. |
| Allow insecure via local IP | On | Admin-toegang via IP zonder certificaatwaarschuwing. |
| Force Multi-Factor Authentication | On (globaal) | MFA verplicht voor elke gebruiker. |
| Session timeout | 48 uur | Demo-gemak: evaluatoren en teamleden hoeven niet elke sessie opnieuw MFA te doen. Productie zou 1 uur of minder zijn. |
| Constellation VPN | Niet geactiveerd | Betaalde functie in v0.22.10; NetBird ZTNA vervangt het volledig (NetBird adverteert `10.0.0.0/8` aan alle peers en geeft zo netwerktoegang tot dc01). |

Cosmos draait als privileged `--network host`-container met de Docker-socket gemount, zodat het poort 80/443 op de host kan binden en de andere service-containers kan beheren. De Docker-daemon op dc01 staat vastgepind op een custom address pool (`192.168.200.0/20`) en MTU 1400 om botsingen met GNS3-segmenten en encapsulation-fragmentatie te vermijden; beide zijn verplicht om de stack te laten werken. De volledige stap-voor-stap-build staat in [Runbook 13: Cosmos](../runbooks/13-cosmos.nl.md).

Wanneer Uptime Kuma via Cosmos Market wordt geïnstalleerd, genereert Cosmos de route-beveiligingsconfig automatisch:

```json
{
  "SmartShield": { "Enabled": true },
  "AuthEnabled": true,
  "BlockCommonBots": true,
  "ThrottlePerMinute": 12000,
  "cosmos-force-network-secured": "true"
}
```

Het label `cosmos-force-network-secured` koppelt applicatietoegang terug aan de netwerklaag: een container zonder Cosmos-route krijgt geen extern netwerkpad, wat ook het mechanisme is achter inbound Shadow IT-indamming op dc01.

---

## Integratiepunten

| Component | Richting | Wat |
|-----------|----------|-----|
| [NetBird](netbird.nl.md) | hangt af van (network admission) | Cosmos is Gate 2; NetBird is Gate 1. Clients bereiken `10.0.0.100` enkel via de NetBird-overlay, dus Cosmos en `*.dc.local` zijn onbereikbaar zonder NetBird. NetBird die `10.0.0.0/8` adverteert maakt dc01 routeerbaar. |
| Unbound op OPNsense (`10.0.0.1`) | DNS-afhankelijkheid | Host overrides mappen `cosmos.dc.local`, `kuma.dc.local`, `gitea.dc.local` → `10.0.0.100` zodat clients de routes kunnen resolven. |
| dc01 `/etc/hosts` | DNS-afhankelijkheid | Cosmos draait in Docker en gebruikt Docker's interne resolver (`127.0.0.11`), die `dc.local` niet kent. Elke `*.dc.local`-route moet ook in dc01's hosts-bestand staan, anders geeft de route een internal server error. |
| NetBird DNS (nameserver-rule) | DNS-afhankelijkheid | Een NetBird nameserver-rule (`10.0.0.1`, match-domein `dc.local`) plus een OPNsense `wt0`-firewall rule voor poort 53 laat overlay-clients `*.dc.local` resolven. |
| Uptime Kuma / Gitea backends | proxiet naar | Reverse-proxied op de host; backend-poorten `3001`/`3000` zijn niet extern gebonden, Cosmos is de enige ingress. |

---

## Bekende problemen / valkuilen

- **Hostname op IP breekt cross-domain OAuth/SSO.** Met de hostname op `10.0.0.100` bouwt Cosmos zijn OAuth-redirect als `https://10.0.0.100/cosmos-ui/openid`. Browser-cookies zijn domain-gebonden, dus een cookie op `10.0.0.100` is niet geldig op `gitea.dc.local`: de `AuthEnabled`-toggle is grijs voor `*.dc.local`-routes. Dit is de grootste architecturale beperking van de installatie. Volledige analyse en het FQDN-migratiepad: [Bevinding: hostname op IP beperkt OAuth](../findings/cosmos-hostname-oauth.nl.md).
- **Nginx moet gemaskerd worden, niet enkel gestopt.** Ubuntu Server kan nginx als dependency binnenhalen; het grijpt poort 80/443 die Cosmos nodig heeft, en `systemctl stop` is niet genoeg omdat het na een reboot herstart. Mask het.
- **`dc.local` is onzichtbaar voor de interne resolver van Cosmos.** De meest verrassende valkuil: een route op `kuma.dc.local` faalt met `lookup kuma.dc.local: no such host` tot de naam aan dc01 `/etc/hosts` is toegevoegd. Voeg elke nieuwe `*.dc.local`-service onmiddellijk aan dat bestand toe.
- **SmartShield heeft geen globale toggle in v0.22.10.** Het wordt per route geconfigureerd, automatisch bij installatie via Cosmos Market; je kunt het niet in één keer voor alle services aanzetten.
- **Niet installeren via Portainer / CasaOS / Unraid-templates.** Die breken de Cosmos-configuratie; gebruik altijd het officiële `docker run`-commando.
- **Constellation VPN is een betaalde functie in v0.22.10.** Probeer het niet te activeren; NetBird vervangt het.
- **Kuma-monitors moeten IP-adressen gebruiken, geen hostnames.** De Kuma-container gebruikt Docker's interne DNS, die `dc.local` niet kent; gebruik `https://10.0.0.100` (Cosmos) om services erachter te monitoren, nooit de interne backend-poort.

Een bredere lijst van build-time valkuilen (nginx masken, daemon.json, DNS search-domein-interferentie, de `wt0`-DNS-firewall rule) staat stap voor stap in [Runbook 13: Cosmos](../runbooks/13-cosmos.nl.md).

---

## Gerelateerd

- [Concept: Application Gateway](../concepts/application-gateway.nl.md)
- [Concept: Zero Trust](../concepts/zero-trust.nl.md)
- [Beslissing: Twee-laags ZTNA (NetBird + Cosmos)](../decisions/cosmos-two-layer-ztna.nl.md)
- [Bevinding: Cosmos hostname op IP beperkt OAuth/SSO](../findings/cosmos-hostname-oauth.nl.md)
- [Runbook 13: Cosmos](../runbooks/13-cosmos.nl.md)
- [Component: NetBird](netbird.nl.md)

---
title: "Concept: WPAD/PAC — Web Proxy Auto-Configuration"
tags: [proxy, wpad, sase, network, squid]
---

# Concept: WPAD/PAC — Web Proxy Auto-Configuration

**Definitie:** Een tweedelig standaard waarbij WPAD (Web Proxy Auto-Discovery) browsers in staat stelt een proxyserver te ontdekken via DNS, en PAC (Proxy Auto-Configuration) het JavaScript-bestand is dat de browser uitvoert om per URL te beslissen of de proxy gebruikt moet worden of rechtstreeks verbinding moet worden gemaakt.

## Hoe het hier van toepassing is

WPAD/PAC is het mechanisme dat elke browseraanvraag op mobile01 via Squid op pop01 routeert — zonder een proxyadres in elke browser hard te coderen. De ontdekkingsketen:

1. Het DNS-zoekdomein van mobile01 is `sandbox.local` (gedistribueerd door NetBird DNS-configuratie)
2. Browser vraagt `wpad.sandbox.local` op → lost op naar mgmt01 overlay-IP `100.70.135.241` (via NetBird Custom DNS Zone)
3. Browser haalt `http://wpad.sandbox.local/wpad.dat` op
4. Browser voert `FindProxyForURL()` uit voor elke URL → geeft `PROXY 100.70.154.79:3128` terug (Squid)
5. Al het niet-interne verkeer stroomt door Squid voor inspectie

**Waarom WPAD/PAC en geen transparante proxy:** Transparante proxy via `pf rdr` kan WireGuard-gerouteerd verkeer op de `wt0`-interface niet onderscheppen — WireGuard is Laag 3 en `pf rdr` ziet vanuit het perspectief van de kernel geen inkomend verkeer op die interface. WPAD/PAC expliciete proxy is geen tijdelijke oplossing; het is het juiste patroon voor overlay-gebaseerde BYOD. Zie [Beslissing: WPAD/PAC vs. transparante proxy](../decisions/wpad-vs-transparent-proxy.md).

**Expliciet vs transparant:**
- Expliciete proxy: client kent het proxyadres en stuurt aanvragen er direct naartoe (HTTP `GET http://...` of HTTPS `CONNECT` gevolgd door de aanvraag)
- Transparante proxy: firewall leidt poort 80/443 stilzwijgend om naar de proxy; client is zich niet bewust

Met expliciete proxymodus kan Squid ook HTTPS verwerken via `CONNECT`-tunnel (en SSL Bump om de inhoud te inspecteren). Transparante HTTPS vereist complexere `pf divert`-regels die ook niet werken op `wt0`.

## Waar het in de stack voorkomt

**[Caddy](../components/caddy.md)** — serveert het PAC-bestand op `http://wpad.sandbox.local/wpad.dat` met het juiste MIME-type (`application/x-ns-proxy-autoconfig`). Draait op het overlay-IP van mgmt01 (`100.70.135.241`).

**[Squid](../components/squid.md)** — de proxy waarnaar het PAC-bestand verwijst: `PROXY 100.70.154.79:3128`. Squid draait de pre-auth include-listener op dit NetBird overlay-IP met volledige ssl-bump-parameters.

**[NetBird](../components/netbird.md)** — biedt twee dingen waarvan WPAD afhankelijk is: (1) het overlaynetwerk dat het IP van mgmt01 bereikbaar maakt vanuit mobile01, en (2) de Custom DNS Zone die `wpad.sandbox.local` oplost naar het overlay-IP van mgmt01.

## Belangrijke onderscheidingen

**PAC-bestand moet DIRECT retourneren voor overlay-IP's** — als het PAC-bestand `PROXY ...` retourneerde voor `100.64.0.0/10` (het NetBird overlay-subnet), zouden browseraanvragen naar de NetBird-beheerconsole via Squid worden gerouteerd, wat een routeringslus zou veroorzaken. Het PAC-bestand moet overlay- en interne subnetten expliciet uitsluiten.

**WPAD dekt alleen browser-HTTP/HTTPS** — applicaties die de systeemproxyinstelling niet lezen (veel niet-browserapps, commandoregeltools) omzeilen Squid volledig. Suricata's netwerklaaginspectie op vtnet0 compenseert gedeeltelijk door afwijkingen in directe verbindingen te detecteren.

**mobile01 is handmatig geconfigureerd** — Windows-instellingen → Proxy → "InstallatieScript gebruiken" → URL: `http://wpad.sandbox.local/wpad.dat`. In een productieomgeving zou dit worden gedistribueerd via Groepsbeleid of MDM.

## Bronnen

- `raw/Doc1_Squid_WPAD_PAC.md` §1–2 (WPAD/PAC-architectuur, PAC-bestand, waarom geen transparante proxy)
- `raw/Verslag19.md` (WPAD/PAC-implementatiesessie)

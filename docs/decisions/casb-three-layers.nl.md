---
title: "Beslissing: Drielaags CASB-architectuur"
tags: [decision, casb, sase, nats, wazuh, squid]
---

# Beslissing: Drielaags CASB-architectuur

**Status:** Geïmplementeerd  
**Datum:** Mei-juni 2026 (Verslag32-Verslag39)

## Context

De SASE-architectuur vereist Cloud Access Security Broker (CASB)-functionaliteit: SWG-verankerde controle over cloud-applicatiegebruik, geïntegreerd met identiteit. Een enkel handhavingsmodel kan niet alle vereiste tijdshorizonten dekken: blokkering op aanvraagtijdstip, post-event auditrespons en real-time cross-component correlatie. De architectuur combineert meerdere handhavingslagen om bruikbare CASB-dekking te bieden binnen de PoC-scope.

## Beslissing

Een drielaags CASB-model is geïmplementeerd, waarbij elke laag een andere tijdshorizon voor handhaving dekt:

**Laag 1, Inline (Squid SWG-pipeline):**
Handhaving op aanvraagtijdstip. Squid verwerkt elk HTTP/HTTPS-verzoek via de ICAP-pipeline en past URL-filtering en identiteitsgebaseerde blokkering toe. Wanneer het verzoek van een gebruiker een geblokkeerde categorie raakt of overeenkomt met een DLP-patroon, wordt het geweigerd voordat het antwoord wordt teruggestuurd. Deze laag werkt op millisecondensniveau: de gebruiker ziet onmiddellijk een blokkeringspagina.

**Laag 2, API-mode (Office 365 Management Activity API + Wazuh + Microsoft Graph API):**
Post-event audithandhaving. Een aangepaste producer (`o365_producer`) pollt de Office 365 Management Activity API op SharePoint-auditevents en publiceert ze naar NATS; Wazuh-regels (de 100600-familie) detecteren vervolgens beleidsschendingen: anonieme link-aanmaak, anyone-scope deellinks en gast-sharing. Bij een schending trekken Wazuh Active Response-scripts (`sharepoint_remediate.sh`, `guest_remediate.sh`) de betreffende link in via de Microsoft Graph API, achter een ENFORCE-gate die standaard op detect-only staat. Deze laag dekt SaaS-activiteit die niet via de proxy verloopt; ze werkt binnen minuten (de Management Activity API heeft tot ~1u latency en geen SLA). De revoke-keten is bewezen tot HTTP-204 via een offline stub; live intrekking tegen een echt OneDrive-bestand wacht op provisioning van een testaccount.

**Laag 3, Real-time (NATS + Control Daemon):**
Cross-component threat scoring en quarantaine. De Control Daemon subscribet op detectie-events op de NATS-bus (`security.alert.>`) en dispatcht op het `producer`-veld. Enkel malware- (c-icap RESPMOD) en DLP-events accruen threat score. Proxy- en IDS-events zijn log-only: ambient proxy-ruis zou vals-positieven accrueren, en Suricata-IDS draagt geen overlay-attributie (dus C2-respons wordt uitgesteld naar Zeek/RITA). De daemon houdt per peer een threat score bij met sliding-window decay. Wanneer een peer de quarantainedrempel overschrijdt, verwijdert de daemon hem uit zijn policy-bearing persona-groepen via de NetBird Groups API. Deny-by-default isoleert hem dan. De verwijdering beperkt zich tot persona-groepen, dus infrastructuur-peers (`Core-Services`) blijven bereikbaar. Deze laag werkt op secondensniveau en is reversibel. De volledige dry-run → ENFORCE → reversibel-keten is end-to-end gevalideerd op testpeer docent1 (V35): een EICAR-malware-event scoorde 80/80, de peer werd binnen enkele seconden in quarantaine geplaatst en verloor connectiviteit, en werd hersteld bij unquarantine.

## Gevolgen

- Elke laag dekt een andere tijdshorizon: aanvraagtijdstip (milliseconden), auditrespons (minuten), cross-component correlatie (seconden).
- In plaats van te scoren tegen het volledige Gartner-CASB-raamwerk (wat het werk zou onderschatten), wordt het model beoordeeld tegen de CASB-rubriek van het project: SWG-verankerde cloud-app-controle. Het dekt de kern goed: **identiteitsgeïntegreerde cloud-app-blokkering** (Laag 1, per persona via de Identity Bridge en Entra ID), **groep-gescoopte policies** (persona-groepen sturen de inline- en real-time-lagen) en **testresultaten met een gevalideerde handhavingsketen** (Laag 3, docent1/EICAR, V35). De dunnere dimensies (context-/risico-/device-gebaseerde blokkering en expliciete bypass-tests) zijn de focus van de verbeteringen hieronder.
- Laag 1 vereist dat de volledige SWG-pipeline (Squid + c-icap + ClamAV + Python DLP) operationeel is.
- Laag 2 vereist de `o365_producer` die de Office 365 Management Activity API pollt, Wazuh-regels plus Active Response, en Microsoft Graph API-toegang voor remediatie; live intrekking vereist daarnaast OneDrive-provisioning van een testaccount.
- Laag 3 vereist dat NATS JetStream, de Control Daemon en NetBird API-toegang gelijktijdig operationeel zijn.
- Het toevoegen van nieuwe CASB-subfuncties betekent het uitbreiden van een van de drie lagen in plaats van een vierde te bouwen. De architectuur is ontworpen om uitbreidbaar te zijn binnen het bestaande model.

## Mogelijke verbeteringen

Nog niet geïmplementeerd. Kandidaat-uitbreidingen die de dunnere rubriek-dimensies (context-based blokkering en bypass-tests) zouden versterken, binnen de bestaande open-source-stack:

- **Tenant Restrictions v2 via SSL-Bump header-injectie.** De organisatie sanctioneert één cloud-suite (M365), dus Squid (doet al SSL Bump) zou `Restrict-Access-To-Tenants` / `Restrict-Access-Context`-headers kunnen injecteren op Microsoft-login-verkeer: zakelijke aanmelding toegestaan, persoonlijke/andere-tenant-aanmelding geblokkeerd. Een echte context-based control in ~8 regels `request_header_add`, die rechtstreeks het rubriek-criterium "dynamische/context-based cloud-app-blokkering" bedient.
- **Shadow-IT-ontdekking uit Squid-accesslogs.** Squid-logs gaan al naar Wazuh; door bestemmingsdomeinen/SNI te classificeren tegen een gecureerde cloud-app-lijst ontstaat een "ontdekte SaaS-apps + risico"-overzicht per gebruiker op het Wazuh-dashboard, zonder nieuwe component. Een gecureerde app→domein-lijst is voor beoordeling verdedigbaarder dan een vendor-catalogus: elke entry is uitlegbaar en bypass-resistentie (SNI + DNS-RPZ + IP) is aantoonbaar.

Zie ook: [Component: Squid](../components/squid.md), [Component: Wazuh](../components/wazuh.md), [Component: Control Daemon](../components/control-daemon.md), [Component: NATS](../components/nats-jetstream.md), [Concept: SASE](../concepts/sase.md)

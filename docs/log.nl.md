---
title: "Wijzigingslog"
tags: [architecture]
---

# Wijzigingslog

Toevoeg-alleen log van wiki-wijzigingen.

---

## 2026-04-10: Initiële ingest (volledige wiki-opbouw)

**Aangemaakte bestanden:**

### Overzicht
- `overview/architecture.md`: Volledige systeemarchitectuur: nodes, segmenten, opsplitsing control/data plane, vier verkeersstromen, inspection pipeline table, trust boundaries

### Componenten
- `components/squid.md`: Expliciete proxy, WPAD/PAC, SSL Bump, URL-filtering, ICAP-orkestratie
- `components/clamav-cicap.md`: Malware-scanning + DLP-laag 1, YARA-regels, StructuredDataDetection
- `components/python-dlp.md`: Upload-DLP ICAP REQMOD, pyicap-patch, multipart-parsing
- `components/suricata.md`: IDS op WAN+LAN, Hyperscan, custom.yaml, validatieresultaten
- `components/ioc2rpz.md`: ioc2rpz + BIND + Unbound RPZ-keten, 71 767 records, twee feeds
- `components/netbird.md`: WireGuard ZTNA-overlay, Zitadel + Entra ID, ACL-beleid, DNS
- `components/caddy.md`: WPAD-server, NetBird TLS-terminator, ioc2rpz GUI-proxy
- `components/gns3.md`: GNS3 op Proxmox, topologie, IP-tabel, snapshots, operationele valkuilen
- `components/vyos.md`: VyOS SD-WAN-gateway (stub, onvoldoende brondetail voor volledige configuratie)

### Concepten
- `concepts/sase.md`: Vijf SASE-pijlers, PoC-mapping, commerciële equivalenten
- `concepts/zero-trust.md`: Driepoortmodel, aanvullende poorten, principemapping
- `concepts/icap.md`: REQMOD vs. RESPMOD, Squid-orkestratie, twee-server-onderbouwing
- `concepts/ssl-bump.md`: HTTPS-interceptie, SASE-PoC-CA, no-bump-lijst, pre-auth ssl-bump
- `concepts/wpad-pac.md`: PAC-bestand ontdekkingsketen, transparante proxy mislukt, expliciete modus
- `concepts/rpz.md`: RPZ-zone transfer chains, NXDOMAIN-afdwinging, DNS-bereik
- `concepts/dlp.md`: Twee-laags DLP, algoritmische validatie, YARA-regelbereik

### Beslissingen
- `decisions/wpad-vs-transparent-proxy.md`: pf rdr mislukt op wt0 → WPAD/PAC
- `decisions/two-layer-dlp.md`: ClamAV multipart-kloof → Python DLP REQMOD
- `decisions/ioc2rpz-vs-unbound-native.md`: Feed-aggregatie → ioc2rpz Docker
- `decisions/bind-tsig-intermediary.md`: Unbound geen TSIG → BIND 9.20 als intermediary
- `decisions/ids-vs-ips.md`: Netmap/virtio-incompatibiliteit → IDS-modus
- `decisions/suricata-wan-lan.md`: wt0 BPF 0 paketten → vtnet0+vtnet1
- `decisions/gns3-vs-eveng.md`: Meerdere-gebruikers-vereiste, QCOW2-formaat → GNS3
- `decisions/zitadel-idp-broker.md`: Quickstart installeert Zitadel, Entra ID als externe IdP
- `decisions/ca-posture-hybrid.md`: Intune-kloof op BYOD → hybride CA + postuur driepoortmodel

### Bevindingen
- `findings/wt0-pf-rdr-limitation.md`
- `findings/pre-auth-ssl-bump-params.md`
- `findings/squid-clearlog-destroys-file.md`
- `findings/stevenblack-incompatible.md`
- `findings/suricata-interface-default-bug.md`
- `findings/suricata-netmap-virtio.md`
- `findings/unbound-no-tsig.md`
- `findings/unbound-config-path.md`
- `findings/ioc2rpz-gui-js-bug.md`
- `findings/pyicap-collections-bug.md`
- `findings/iptables-forward-ordering.md`
- `findings/netbird-primary-nameserver.md`
- `findings/netbird-config-zero-bytes.md`
- `findings/docker-volume-recreation.md`

### Index en log
- `index.md`: Volledige catalogus, snelle naslag (poorten, IP's, poortstatus)
- `log.md`: Dit bestand

**Verwerkte brondocumenten:** Verslag18–27 (10 sessieverslagen), Doc1–Doc7 (7 implementatiedocumenten), SASE_Architectuur_Overzicht.md (1 architectuuroverzicht)

**Totaal aantal aangemaakte pagina's:** 37

---

## 2026-06-03: Grote update (V28–V44 broningest, nieuwe componenten, event-driven handhaving)

**Opgelost probleem:** De wiki was gebaseerd op V18–V27 en Doc1–Doc7. Sindsdien zijn 17 nieuwe verslagen (V28–V44) en 10 nieuwe implementatiedocumenten geproduceerd, die introduceerden: Identity Bridge, NATS JetStream event bus, Control Daemon, Wazuh SIEM, Zitadel Actions, Zero Trust Branch-model, CASB drielaags-architectuur, en volledige Gate 1+2 implementatie.

**Scopewijzigingen:**
- BYOD → beheerde Windows-apparaten (Intune-beheerd, Entra joined) per lectoraatmandaat R11
- SD-WAN geschrapt → Zero Trust Branch-model met VyOS SASE Gateway
- Gates 1+2 van gepland naar operationeel (5 CA-beleid, Intune-conformiteit)
- CASB uitgebreid van gedeeltelijke inline naar drielaags model (inline + API + real-time)

**Aangemaakte bestanden (42 nieuwe pagina's):**

### Componenten (10 bestanden)
- `components/identity-bridge.md` + `.nl.md`: FastAPI overlay-IP → Entra ID-persona groep
- `components/nats-jetstream.md` + `.nl.md`: Centrale event bus die detectiesilo's verbindt
- `components/control-daemon.md` + `.nl.md`: Threat scoring + quarantaine via NetBird API
- `components/wazuh.md` + `.nl.md`: SIEM met NATS-forwarder + M365 Active Response
- `components/zitadel.md` + `.nl.md`: OIDC IdP-broker met twee Zitadel Actions

### Concepten (2 bestanden)
- `concepts/identity-flow.md` + `.nl.md`: Volledige identiteitsketen

### Beslissingen (14 bestanden)
- 7 beslissingen × 2 talen: netbird-service-pat, nats-accounts-auth, groupsync-pad-b, casb-three-layers, managed-devices-scope, zt-sdwan-branch, control-daemon-scope

### Bevindingen (16 bestanden)
- 8 bevindingen × 2 talen: netbird-issue-3127, squid-overlay-bind-race, overlay-ip-instability, nats-store-dir, wazuh-cpu-glibc, wazuh-dashboard-airgate, netbird-jwt-allow-groups-lockout, dc-lan-isolation-route-acl

### Testen (2 bestanden)
- `testing/attack-scenarios.md` + `.nl.md`: 22 demovalidatie-scenario's per SASE-pijler

### Runbooks (8 bestanden)
- `runbooks/08-groupsync.md` + `.nl.md`, `runbooks/09-identity-bridge.md` + `.nl.md`, `runbooks/10-nats-jetstream.md` + `.nl.md`, `runbooks/11-wazuh.md` + `.nl.md`

**Bijgewerkte bestanden (30+ bestanden):** Architectuur, index, concepten (sase, zero-trust), 6 bestaande componenten (NATS-integratie), VyOS volledige vervanging, beslissingen (sdwan-descoped, ca-posture-hybrid, zitadel-idp-broker), acceptatietests (F3/F10/F11/F15 + T-A10–T-A13), runbook-07, runbook-index, tags.

**Verwerkte brondocumenten:** V28–V44 (17 verslagen), 10 implementatiedocumenten.

**Totaal nieuwe pagina's:** 42. **Totaal bijgewerkte pagina's:** 30+.

---

## 2026-06-04: Raw → Wiki eenrichtings-consistentieaudit (volledige doorloop)

**Doel:** Elke wikipagina in één richting gelijkstellen aan de bronnen in `raw/` (`raw/` is de
enige bron van waarheid); enkel de wiki is gewijzigd. Beide talen gecorrigeerd en EN↔NL-divergentie
als defect behandeld. Scope = enkel correctheid (geen redactionele inkorting, geen
hernoemingen/verplaatsingen/slug-wijzigingen).

**Toegepast gezagsmodel:** verslag-bevestigde toestand = grondwaarheid; een later verslag vervangt
een eerder; correctienotities overschrijven hun bovenliggend plandocument maar een verslag wint nog
steeds; addenda / architectuurdocumenten / Doc1–7 = intentie vóór implementatie (enkel rationale +
gatenvulling). Dit keert de regel om waaronder de wiki oorspronkelijk werd gebouwd (zie CLAUDE.md-fix
hieronder), wat verklaart waarom de grootste driftklasse pagina's waren die een plan/addendum hadden
gevolgd waar een verslag later van afweek.

**Geauditeerde domeinen (alle 12 + overkoepelende Fase E):** DNS-Threat-Intel · SWG/Proxy ·
Malware+DLP · IDS · Lab/Virtualisatie · ZTNA-Overlay · Context-Aware/CA · GroupSync ·
Identity-Bridge/Control-Daemon · NATS/JetStream · CASB+Wazuh · ZT-SD-WAN · Fase E
(overview/architecture, concepten, index, tags, sectie-indexen, testing/*).

**Bevindingen per klasse (~88 afzonderlijke bevindingen over 86 logische pagina's × EN+NL):**
- **feitelijk (~28):** verkeerde app-reg-GUID (`cebe0d74…`→`11803ee8…`), Identity-Bridge poll-doel
  (`HTTPS overlay`→`http://management:80`), NATS-versie (`2.14`→`2.14.1-alpine`), datumdrift Suricata/DNS
  (april→maart 2026), Control-Daemon-datum (mei→juni), hoofdletters groepsnaam
  (`2ITcsc1A-`→`2ITCSC1A-` over 4 pagina's), VyOS-versie (`1.4 rolling`→`rolling 2026.02.16`),
  sitepc01 Site-LAN-IP (`172.16.10.50`→`172.16.10.10`), e.a.
- **verouderd-vervangen (~13):** NetBird-groepen/ACL (SASE-* → personamodel + één
  `Personas-to-Core-Services`-beleid, V34); CA-posture (NetBird-posture → Intune-conformiteit Gate 2,
  V40); sitepc01 (`geen OS` → Tiny11 operationeel, V43/V44); groepstabellen runbook-07/08/09.
- **status-drift (~10):** Wazuh CASB live-revoke (werkend → detect-only HTTP-204-stub, live in
  afwachting); VyOS QoS+failover (gepland/geschrapt → geïmplementeerd, V43); GroupSync #5399 Dex-gotcha
  (live → niet van toepassing, V30.17); acceptatietests F4/F8 (zie onder).
- **inconsistentie / EN↔NL-divergentie (~20):** dominant NL-patroon was weggevallen inhoud (NL-pagina's
  misten blokken die EN wel had); plus één zelfcontradictie (`concepts/identity-flow` benoemde de
  mechanismekeuze JWT-vs-IdP-Sync foutief als "Pad A/B", wat raw voorbehoudt voor prefixafhandeling).
- **ongegrond-en-fout (~10):** `components/identity-bridge` verzon `/member`, `children-max=5`,
  `identity.login`/`identity.group_change`; `findings/wazuh-dashboard-airgate` verzon een
  `api.wazuh.com`-URL + "eeuwige spinner"; `-p 953` op `rndc` (×4 DNS-pagina's); NATS-subject
  `control.quarantine`.
- **link (~7):** ontbrekende Related-runbook-links toegevoegd tijdens het bewerken. Finale linkaudit
  over alle 172 bestanden (1.259 interne `.md`-links) = **0 gebroken**.

**Belangrijkste resoluties (gezagsmodel in actie):**
- **GroupSync Pad A/B:** Pad B is verscheept (V31/V34) en is een **fail-closed allowlist die de
  Entra-weergavenaam-STRING → schone NetBird-naam mapt** (`2ITCSC1A-Studenten`→`Studenten`),
  niet-gematchte groepen worden stil gedropt. Netto-effect strips de prefix maar strikter dan een
  blinde strip. Entra-beveiligingsgroep-hoofdletters zijn **all-caps `2ITCSC1A-`** (verslagen V34/V40
  vervangen de kleine letters `2ITcsc1A-` uit de correctienotitie). De oorspronkelijke "GUID-gesleutelde"
  fout van de wiki ÉN een tussentijdse "generieke prefix-strip"-overcorrectie gecorrigeerd.
- **Quarantaine breekt actieve flow = GETEST, eerlijk geherkaderd:** V35.14 dry-run op `docent1`:
  EICAR scoorde 80/80, peer in quarantaine **binnen enkele seconden** (persona-groepen gestript →
  deny-by-default), connectiviteit verloren, hersteld bij un-quarantine. Voorgesteld als getest met
  timing **"binnen enkele seconden"** (niet "milliseconden") en ZONDER de niet-onderbouwde claim "pagina
  stierf mid-load". ENFORCE=false tot Sessie 11.
- **CASB-demo-SID:** `100201` → **`100601`** (`AnonymousLinkCreated`, V39 sandbox-basisfamilie 100600).
  De `100500/100501` uit het seed zijn de bus-regels van het Marnix-team, niet de sandbox-CASB-regels.
- **Acceptatietests F4/F8 (Datacenter Access / Firewall-segmentatie):** geherkaderd als
  **gevalideerd-daarna-verwijderd**: het ✅ resultaat bij opbouw behouden, met huidige-toestand-kadering
  dat het DC-LAN-over-overlay-pad (`Internal-DC` Network + `Datacenter Access`-beleid) verwijderd is in de
  V34-personamigratie en uitgesteld; de negatieve-isolatiehelft van F8 blijft geldig. Toegepast op F4, F8
  en F15 stap 6 + telling, EN+NL.

**CLAUDE.md-gezagsregel gecorrigeerd:** de projectinstructie "het implementatiedocument wint van het
verslag voor de huidige toestand" is omgekeerd naar het gecorrigeerde model: **verslag-bevestigde
toestand is grondwaarheid; addenda / implementatiedocumenten / plandocumenten zijn intentie vóór
implementatie en overschrijven nooit een verslag voor de huidige toestand.** Dit is de grondoorzaak van
de systematische drift die deze audit corrigeerde.

**Gewijzigde bestanden:** wikipagina's over alle domeinen (EN+NL) volgens de per-domein-tabellen in
`.agent-memory/audit-tracker.md`; `CLAUDE.md` (gezagsregel); `log.md` + `log.nl.md` (deze entry).
Geen `raw/`-bestanden gewijzigd. Geen hernoemingen, verplaatsingen, slug- of frontmatter-wijzigingen.

---

## 2026-06-05: BYOD + vertaalopschoning + grondige heraudit

**Doel:** Alle resterende verouderde "BYOD"-verwijzingen in huidige-staat-beschrijvingen herstellen,
oververtaalde Nederlandse technische termen corrigeren (~181 instanties), daarna een volledige
multi-agent heraudit van elke wikipagina tegen de raw verslagen uitvoeren ter verificatie vóór deployment.

### Fase 1: BYOD-opschoning (40 verouderde verwijzingen → 0)
Alle huidige-staat "BYOD client" / "BYOD persona"-beschrijvingen vervangen door correcte termen
("overlay client", "managed Windows client", enz.) over 55 EN+NL-bestanden.

### Fase 2: Oververtaalde Nederlandse termen (~181 fixes)
Engelse technische termen die onnodig vertaald waren naar het Nederlands hersteld:
`eindknooppunt` → exit node, `beleidsregels` → policies, `handtekeningen` → signatures,
`gebeurtenissen` → events, `drempelwaarde` → threshold, `naamserver` → nameserver,
`zonetransfer` → zone transfer, `verbindingspooling` → connection pooling,
`Luisterpoort` → Listen Port, `Doelbronnen` → Target resources, `Toewijzingen` → Assignments,
`Voorwaarden` → Conditions, `Toegangscontroles` → Access controls,
`Aanmeldingslogboeken` → Sign-in logs, en meer.

### Fase 3: Volledige heraudit (4 domeinagenten + 1 onafhankelijke Opus-auditor)
Vier parallelle verificatieagenten dekten alle 12 domeinen + cross-cutting pagina's, elke wikiclaim
gecontroleerd tegen de raw verslagen. Een onafhankelijke auditor (Opus-model) kruiste vervolgens
de bevindingen van de domeinagenten.

**Resultaten domeinagenten (gecombineerd 37+ 11 + 12 + 30 checks):**
- DNS+SWG+DLP+IDS: ALLES GESLAAGD
- ZTNA+CA+GroupSync: ALLES GESLAAGD (37/37)
- Identity+NATS+CASB+Wazuh: GESLAAGD met 3 kleine hiaten (opgelost)
- SD-WAN+Lab+CrossCutting: 4 discrepanties (opgelost)

**Fixes toegepast vanuit audit:**
- `decisions/sdwan-descoped.md` + `.nl.md`: F15 stap 8 is gevalideerd (niet N.v.t.); VyOS is
  SASE-gateway met QoS (niet "minimaal, enkel NAT"); onderbouwing gecorrigeerd (QoS herimplementeerd)
- `overview/architecture.nl.md`: ontbrekende "SWG-identiteitslaag" trust boundary rij toegevoegd (EN/NL pariteit)
- `decisions/nats-accounts-auth.md` + `.nl.md`: `$JS.ACK.>` toegevoegd naast `$JS.API.>` voor consumers (V32)
- `components/control-daemon.md` + `.nl.md`: expliciete scoring-gewichten `malware=80`, `dlp_match=30` + V35-testreferentie
- `components/nats-jetstream.md` + `.nl.md`: 5 JetStream-streamstabel toegevoegd (V32.13)
- **Onafhankelijke auditor vals positief (teruggedraaid):** auditor markeerde `security.alert.ids` → `.ips`
  op basis van V34's momentopname, maar de live nats.conf op de server bevestigt dat `.ids` de huidige
  subjectnaam is; de config werd bijgewerkt na V34. Wijziging werd toegepast en vervolgens teruggedraaid.

**Samenvatting onafhankelijke auditor: 30 GESLAAGD, 1 vals positief (teruggedraaid).** De gemelde
NATS-subject-discrepantie was gebaseerd op V34's momentopname; de live serverconfiguratie bevestigt `.ids`.

**Verificatiesweep-resultaten:**
- Oververtaalde termen: 0 resterend
- BYOD in huidige staat: 0 (28 resterend allemaal in historisch/evolutie-framing)
- Gelekte secrets: 0
- EN↔NL pariteit: geverifieerd over 5 paginaparen
- Alle IP's, poorten, versies, groepsnamen consistent cross-page

**Gewijzigde bestanden:** 63 wikipagina's (EN+NL). Geen `raw/`-bestanden gewijzigd.

---

## 2026-06-05: Wazuh dashboard airgate finding herschreven (nieuw brondocument: Wazuh_DB_Fix.md)

**Nieuw brondocument:** `raw/Wazuh_DB_Fix.md` (v1.0, 2 juni 2026), versie-herstel runbook voor de Wazuh-stack na een `down -v`-incident + versie-skew. Gevalideerd op 2 juni 2026.

**Oorzaak gecorrigeerd:** De bestaande finding (`findings/wazuh-dashboard-airgate.md`) had de verkeerde root cause: de air-gap CTI-500 (`GET /manager/version/check`) werd aangewezen als oorzaak van de "Status: Offline"-redirect. Het nieuwe brondocument onthult dat de redirect wordt veroorzaakt door een **lege manager-UUID** in `global.db` (gewist door `docker compose down -v`). De CTI-500 is een apart, cosmetisch, non-blocking probleem in Wazuh 4.14.5+ (fix #8130).

**Statuswijziging:** Finding-status bijgewerkt van "bekende beperking / workaround" naar **opgelost** (2 juni 2026), via in-place bump naar 4.14.5 GA (zonder `-v`), waarbij de bestaande UUID bewaard bleef.

**Audit:** Onafhankelijke Opus auditor-agent heeft alle geplande wijzigingen geverifieerd t.o.v. `Wazuh_DB_Fix.md` vóór de edits werden uitgevoerd. 8 geplande edits: alle PASS. 2 extra gemiste bestanden geïdentificeerd door de auditor en toegevoegd aan de scope.

**Gewijzigde bestanden (10):**
- `findings/wazuh-dashboard-airgate.md` + `.nl.md`: volledige herschrijving: correcte root cause (lege UUID), twee structurele regels, oplossing, bijgewerkte lessen
- `components/wazuh.md` + `.nl.md`: dashboard-status bijgewerkt naar "volledig operationeel"; known-issues-vermelding gecorrigeerd
- `runbooks/11-wazuh.md` + `.nl.md`: gotcha-callout bijgewerkt (opgelost); checklist-item gecorrigeerd (app-modules niet langer gegate)
- `index.md` + `.nl.md`: finding-blurb gecorrigeerd (was "hostnaam-resolutieprobleem", toont nu lege UUID als oorzaak)

**Verwerkte brondocumenten:** `Wazuh_DB_Fix.md` (1 versie-herstel runbook). Geen `raw/`-bestanden gewijzigd.

---

## 2026-06-05: NL-stijlcorrecties (fase 1 + 2)

**Fase 1: calques, samenstellingen, zinsstructuur (34 NL-bestanden):**
Nederlandse calque-fixes over alle `.nl.md`-bestanden: werkwoord-calques (bevragen→queryen, oplossen→resolven in DNS-context, doorsturen→forwarden), geforceerde samenstellingen (uitgangsknoop→exit node, beveiligingsstapel→security stack), Engelse participium-constructies verwijderd, passief→actief herschreven. Gebaseerd op empirische NL-stijlreferentie afgeleid uit het werkelijke technische taalgebruik van de auteur.

**Fase 2: em dash-verwijdering (82 NL-bestanden, ~900 instanties):**
Alle em dashes in lopende tekst vervangen door contextueel gepaste Nederlandse leestekens: haakjes (voorkeur voor tussenwerpsels), dubbele punten, punten, komma's of puntkomma's. En dashes in numerieke bereiken (F1–F15) behouden. Em dashes in codeblokken behouden. Lege tabelcel-placeholders genormaliseerd van em dash naar `n.v.t.`. `attack-scenarios.nl.md` dubbel-koppelteken (`--`) patronen ook gecorrigeerd voor consistentie.

**Audit:** Onafhankelijke Opus auditor-agents verifieerden beide fases voor uitvoering. Fase 1: 10/10 PASS. Fase 2: 18/18 steekproefbestanden PASS, 1 bestaand probleem (attack-scenarios.nl.md) gedetecteerd en opgelost.

**Gewijzigde bestanden:** 82 NL-wikipagina's. Geen EN-pagina's of `raw/`-bestanden gewijzigd.

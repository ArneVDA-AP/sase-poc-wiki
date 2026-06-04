---
title: "Wijzigingslog"
tags: [architecture]
---

# Wijzigingslog

Toevoeg-alleen log van wiki-wijzigingen.

---

## 2026-04-10 — Initiële ingest (volledige wiki-opbouw)

**Aangemaakte bestanden:**

### Overzicht
- `overview/architecture.md` — Volledige systeemarchitectuur: knooppunten, segmenten, opsplitsing control/data plane, vier verkeersstromen, inspectiepijplijn-tabel, vertrouwensgrenzen

### Componenten
- `components/squid.md` — Expliciete proxy, WPAD/PAC, SSL Bump, URL-filtering, ICAP-orkestratie
- `components/clamav-cicap.md` — Malware-scanning + DLP-laag 1, YARA-regels, StructuredDataDetection
- `components/python-dlp.md` — Upload-DLP ICAP REQMOD, pyicap-patch, multipart-parsing
- `components/suricata.md` — IDS op WAN+LAN, Hyperscan, custom.yaml, validatieresultaten
- `components/ioc2rpz.md` — ioc2rpz + BIND + Unbound RPZ-keten, 71 767 records, twee feeds
- `components/netbird.md` — WireGuard ZTNA-overlay, Zitadel + Entra ID, ACL-beleid, DNS
- `components/caddy.md` — WPAD-server, NetBird TLS-terminator, ioc2rpz GUI-proxy
- `components/gns3.md` — GNS3 op Proxmox, topologie, IP-tabel, snapshots, operationele valkuilen
- `components/vyos.md` — VyOS SD-WAN-gateway (stub — onvoldoende brondetail voor volledige configuratie)

### Concepten
- `concepts/sase.md` — Vijf SASE-pijlers, PoC-mapping, commerciële equivalenten
- `concepts/zero-trust.md` — Driepoortmodel, aanvullende poorten, principemapping
- `concepts/icap.md` — REQMOD vs. RESPMOD, Squid-orkestratie, twee-server-onderbouwing
- `concepts/ssl-bump.md` — HTTPS-interceptie, SASE-PoC-CA, no-bump-lijst, pre-auth ssl-bump
- `concepts/wpad-pac.md` — PAC-bestand ontdekkingsketen, transparante proxy mislukt, expliciete modus
- `concepts/rpz.md` — RPZ-zonetransferketens, NXDOMAIN-afdwinging, DNS-bereik
- `concepts/dlp.md` — Twee-laags DLP, algoritmische validatie, YARA-regelbereik

### Beslissingen
- `decisions/wpad-vs-transparent-proxy.md` — pf rdr mislukt op wt0 → WPAD/PAC
- `decisions/two-layer-dlp.md` — ClamAV multipart-kloof → Python DLP REQMOD
- `decisions/ioc2rpz-vs-unbound-native.md` — Feed-aggregatie → ioc2rpz Docker
- `decisions/bind-tsig-intermediary.md` — Unbound geen TSIG → BIND 9.20 als tussenpersoon
- `decisions/ids-vs-ips.md` — Netmap/virtio-incompatibiliteit → IDS-modus
- `decisions/suricata-wan-lan.md` — wt0 BPF 0 paketten → vtnet0+vtnet1
- `decisions/gns3-vs-eveng.md` — Meerdere-gebruikers-vereiste, QCOW2-formaat → GNS3
- `decisions/zitadel-idp-broker.md` — Quickstart installeert Zitadel, Entra ID als externe IdP
- `decisions/ca-posture-hybrid.md` — Intune-kloof op BYOD → hybride CA + postuur driepoortmodel

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
- `index.md` — Volledige catalogus, snelle naslag (poorten, IP's, poortstatus)
- `log.md` — Dit bestand

**Verwerkte brondocumenten:** Verslag18–27 (10 sessieverslagen), Doc1–Doc7 (7 implementatiedocumenten), SASE_Architectuur_Overzicht.md (1 architectuuroverzicht)

**Totaal aantal aangemaakte pagina's:** 37

---

## 2026-06-03 — Grote update: V28–V44 broningest, nieuwe componenten, event-driven handhaving

**Opgelost probleem:** De wiki was gebaseerd op V18–V27 en Doc1–Doc7. Sindsdien zijn 17 nieuwe verslagen (V28–V44) en 10 nieuwe implementatiedocumenten geproduceerd, die introduceerden: Identity Bridge, NATS JetStream event bus, Control Daemon, Wazuh SIEM, Zitadel Actions, Zero Trust Branch-model, CASB drielaags-architectuur, en volledige Gate 1+2 implementatie.

**Scopewijzigingen:**
- BYOD → beheerde Windows-apparaten (Intune-beheerd, Entra joined) per lectoraatmandaat R11
- SD-WAN geschrapt → Zero Trust Branch-model met VyOS SASE Gateway
- Gates 1+2 van gepland naar operationeel (5 CA-beleid, Intune-conformiteit)
- CASB uitgebreid van gedeeltelijke inline naar drielaags model (inline + API + real-time)

**Aangemaakte bestanden (42 nieuwe pagina's):**

### Componenten (10 bestanden)
- `components/identity-bridge.md` + `.nl.md` — FastAPI overlay-IP → Entra ID-persona groep
- `components/nats-jetstream.md` + `.nl.md` — Centrale event bus die detectiesilo's verbindt
- `components/control-daemon.md` + `.nl.md` — Threat scoring + quarantaine via NetBird API
- `components/wazuh.md` + `.nl.md` — SIEM met NATS-forwarder + M365 Active Response
- `components/zitadel.md` + `.nl.md` — OIDC IdP-broker met twee Zitadel Actions

### Concepten (2 bestanden)
- `concepts/identity-flow.md` + `.nl.md` — Volledige identiteitsketen

### Beslissingen (14 bestanden)
- 7 beslissingen × 2 talen: netbird-service-pat, nats-accounts-auth, groupsync-pad-b, casb-three-layers, managed-devices-scope, zt-sdwan-branch, control-daemon-scope

### Bevindingen (16 bestanden)
- 8 bevindingen × 2 talen: netbird-issue-3127, squid-overlay-bind-race, overlay-ip-instability, nats-store-dir, wazuh-cpu-glibc, wazuh-dashboard-airgate, netbird-jwt-allow-groups-lockout, dc-lan-isolation-route-acl

### Testen (2 bestanden)
- `testing/attack-scenarios.md` + `.nl.md` — 22 demovalidatie-scenario's per SASE-pijler

### Runbooks (8 bestanden)
- `runbooks/08-groupsync.md` + `.nl.md`, `runbooks/09-identity-bridge.md` + `.nl.md`, `runbooks/10-nats-jetstream.md` + `.nl.md`, `runbooks/11-wazuh.md` + `.nl.md`

**Bijgewerkte bestanden (30+ bestanden):** Architectuur, index, concepten (sase, zero-trust), 6 bestaande componenten (NATS-integratie), VyOS volledige vervanging, beslissingen (sdwan-descoped, ca-posture-hybrid, zitadel-idp-broker), acceptatietests (F3/F10/F11/F15 + T-A10–T-A13), runbook-07, runbook-index, tags.

**Verwerkte brondocumenten:** V28–V44 (17 verslagen), 10 implementatiedocumenten.

**Totaal nieuwe pagina's:** 42. **Totaal bijgewerkte pagina's:** 30+.

---

## 2026-06-04 — Raw → Wiki eenrichtings-consistentieaudit (volledige doorloop)

**Doel:** Elke wikipagina in één richting gelijkstellen aan de bronnen in `raw/` — `raw/` is de
enige bron van waarheid; enkel de wiki is gewijzigd. Beide talen gecorrigeerd en EN↔NL-divergentie
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
- **feitelijk (~28)** — verkeerde app-reg-GUID (`cebe0d74…`→`11803ee8…`), Identity-Bridge poll-doel
  (`HTTPS overlay`→`http://management:80`), NATS-versie (`2.14`→`2.14.1-alpine`), datumdrift Suricata/DNS
  (april→maart 2026), Control-Daemon-datum (mei→juni), hoofdletters groepsnaam
  (`2ITcsc1A-`→`2ITCSC1A-` over 4 pagina's), VyOS-versie (`1.4 rolling`→`rolling 2026.02.16`),
  sitepc01 Site-LAN-IP (`172.16.10.50`→`172.16.10.10`), e.a.
- **verouderd-vervangen (~13)** — NetBird-groepen/ACL (SASE-* → personamodel + één
  `Personas-to-Core-Services`-beleid, V34); CA-posture (NetBird-posture → Intune-conformiteit Gate 2,
  V40); sitepc01 (`geen OS` → Tiny11 operationeel, V43/V44); groepstabellen runbook-07/08/09.
- **status-drift (~10)** — Wazuh CASB live-revoke (werkend → detect-only HTTP-204-stub, live in
  afwachting); VyOS QoS+failover (gepland/geschrapt → geïmplementeerd, V43); GroupSync #5399 Dex-gotcha
  (live → niet van toepassing, V30.17); acceptatietests F4/F8 (zie onder).
- **inconsistentie / EN↔NL-divergentie (~20)** — dominant NL-patroon was weggevallen inhoud (NL-pagina's
  misten blokken die EN wel had); plus één zelfcontradictie (`concepts/identity-flow` benoemde de
  mechanismekeuze JWT-vs-IdP-Sync foutief als "Pad A/B", wat raw voorbehoudt voor prefixafhandeling).
- **ongegrond-en-fout (~10)** — `components/identity-bridge` verzon `/member`, `children-max=5`,
  `identity.login`/`identity.group_change`; `findings/wazuh-dashboard-airgate` verzon een
  `api.wazuh.com`-URL + "eeuwige spinner"; `-p 953` op `rndc` (×4 DNS-pagina's); NATS-subject
  `control.quarantine`.
- **link (~7)** — ontbrekende Related-runbook-links toegevoegd tijdens het bewerken. Finale linkaudit
  over alle 172 bestanden (1.259 interne `.md`-links) = **0 gebroken**.

**Belangrijkste resoluties (gezagsmodel in actie):**
- **GroupSync Pad A/B:** Pad B is verscheept (V31/V34) en is een **fail-closed allowlist die de
  Entra-weergavenaam-STRING → schone NetBird-naam mapt** (`2ITCSC1A-Studenten`→`Studenten`),
  niet-gematchte groepen worden stil gedropt — netto-effect strips de prefix maar strikter dan een
  blinde strip. Entra-beveiligingsgroep-hoofdletters zijn **all-caps `2ITCSC1A-`** (verslagen V34/V40
  vervangen de kleine letters `2ITcsc1A-` uit de correctienotitie). De oorspronkelijke "GUID-gesleutelde"
  fout van de wiki ÉN een tussentijdse "generieke prefix-strip"-overcorrectie gecorrigeerd.
- **Quarantaine breekt actieve flow = GETEST, eerlijk geherkaderd:** V35.14 dry-run op `docent1` —
  EICAR scoorde 80/80, peer in quarantaine **binnen enkele seconden** (persona-groepen gestript →
  deny-by-default), connectiviteit verloren, hersteld bij un-quarantine. Voorgesteld als getest met
  timing **"binnen enkele seconden"** (niet "milliseconden") en ZONDER de niet-onderbouwde claim "pagina
  stierf mid-load". ENFORCE=false tot Sessie 11.
- **CASB-demo-SID:** `100201` → **`100601`** (`AnonymousLinkCreated`, V39 sandbox-basisfamilie 100600).
  De `100500/100501` uit het seed zijn de bus-regels van het Marnix-team, niet de sandbox-CASB-regels.
- **Acceptatietests F4/F8 (Datacenter Access / Firewall-segmentatie):** geherkaderd als
  **gevalideerd-daarna-verwijderd** — het ✅ resultaat bij opbouw behouden, met huidige-toestand-kadering
  dat het DC-LAN-over-overlay-pad (`Internal-DC` Network + `Datacenter Access`-beleid) verwijderd is in de
  V34-personamigratie en uitgesteld; de negatieve-isolatiehelft van F8 blijft geldig. Toegepast op F4, F8
  en F15 stap 6 + telling, EN+NL.

**CLAUDE.md-gezagsregel gecorrigeerd:** de projectinstructie "het implementatiedocument wint van het
verslag voor de huidige toestand" is omgekeerd naar het gecorrigeerde model — **verslag-bevestigde
toestand is grondwaarheid; addenda / implementatiedocumenten / plandocumenten zijn intentie vóór
implementatie en overschrijven nooit een verslag voor de huidige toestand.** Dit is de grondoorzaak van
de systematische drift die deze audit corrigeerde.

**Gewijzigde bestanden:** wikipagina's over alle domeinen (EN+NL) volgens de per-domein-tabellen in
`.agent-memory/audit-tracker.md`; `CLAUDE.md` (gezagsregel); `log.md` + `log.nl.md` (deze entry).
Geen `raw/`-bestanden gewijzigd. Geen hernoemingen, verplaatsingen, slug- of frontmatter-wijzigingen.

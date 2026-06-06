---
title: "Testblok: transparante proxy en enforcement"
tags: [testing, tproxy, proxy, enforcement, intune, sase]
---

# Testblok: transparante proxy en enforcement

**Scope:** Diagnostische validatie van het transparante-proxyonderzoek (Optie C verworpen, Optie D architectureel onderbouwd) plus de managed-device enforcement-variant die live op de sandbox draait.  
**Bron:** TransProxy_Verslag02, Addendum F v3, Verslag41.

---

## Positie binnen de acceptatietests

Dit testblok valt buiten de oorspronkelijke F1-F15-reeks maar levert bewijs voor dezelfde rubric-criteria. De tests zijn verdeeld in twee categorieën:

1. **T-TP-serie** (Transparent Proxy): diagnostische tests die de architectuurkeuze onderbouwen. Uitgevoerd op de `.11/.17`-omgeving (de parallelle stack), niet herhaald op de sandbox (Sessie 9 gedeprioriteerd).
2. **T-ME-serie** (Managed Enforcement): de endpoint-enforcement-variant die op de sandbox live is (V41). Dit is het productiepad voor beheerde devices.

Het volledige component staat op [Transparante proxy: TPROXY op linuxpop01](../components/transparent-proxy.md).

---

## Algemene status

| Test | Naam | Status |
|---|---|---|
| **T-TP1** | Source-IP bewaard na masquerade OFF | ✅ Gevalideerd (`.11`-omgeving) |
| **T-TP2** | pf rdr matcht overlay-verkeer op vtnet0 | ✅ Gevalideerd (`.11`-omgeving) |
| **T-TP3** | `SYN_RCVD`-hang: return-pad asymmetrisch bij source-IP-behoud | ✅ Gevalideerd (`.11`-omgeving); bewijst dat Optie C faalt |
| **T-TP4** | Masquerade ON maskeert het probleem | ✅ Gevalideerd (`.11`-omgeving); verklaart waarom eerdere tests slaagden |
| **T-TP5** | TPROXY source-IP-behoud (Optie D, test 5) | ✖ Niet uitgevoerd; Sessie 9 gedeprioriteerd; architectureel onderbouwd (`IP_TRANSPARENT` kernel-eigenschap) |
| **T-TP6** | Symmetrisch return-pad (Optie D, test 10) | ✖ Niet uitgevoerd; Sessie 9 gedeprioriteerd; structureel gegarandeerd (interceptie en terminatie op dezelfde node) |
| **T-ME1** | Intune proxy-CSP: bump + overlay-`%SRC` in access.log | ✅ Gevalideerd (sandbox, V41) |
| **T-ME2** | Intune firewall-CSP: QUIC→TCP-fallback + SSL Bump | ✅ Gevalideerd (sandbox, V41) |
| **T-ME3** | Intune firewall-CSP: tunnel + web + MDM intact | ✅ Gevalideerd (sandbox, V41) |
| **T-ME4** | DLP-blokkade end-to-end via afgedwongen proxy | ✅ Gevalideerd (sandbox, V41) |

> Optie D (T-TP5/T-TP6) is **ontworpen, niet getest**. De diagnostiek (T-TP1-T-TP4) is empirisch gevalideerd op de parallelle stack; de twee TPROXY-tests zijn niet uitgevoerd omdat linuxpop01 niet op de sandbox bestaat en Sessie 9 gedeprioriteerd is. Het verwachte resultaat volgt uit een gedocumenteerde kernel-eigenschap, niet uit een meting.

---

## Dekking per rubric-criterium

### SWG: Traffic routing (geen local breakout)

**Rubric-criterium:** volledig geforceerd via SWG.

| Test | Wat het bewijst |
|---|---|
| T-TP3 | Diagnostisch bewijs *waarom* gecentraliseerde pf rdr faalt: architecturele onderbouwing van de keuze voor TPROXY of endpoint-enforcement. |
| T-ME1 | Intune-pushed proxy is systeembreed voor alle WinINET-respecterende apps; niet door de user terug te draaien (non-admin). |
| T-ME2 | QUIC-verkeer wordt geblokkeerd door de firewall-CSP; de browser valt terug op TCP/443 → Squid bumpt het alsnog. |
| T-ME3 | Firewall-regels breken geen legitiem verkeer: NetBird-tunnel, webbrowsing via de proxy en MDM-check-in blijven intact. |

### SWG: Malware-inspectie (DPI)

**Rubric-criterium:** DPI + TLS-decryptie + malware-detectie.

| Test | Wat het bewijst |
|---|---|
| T-ME4 | DLP-ICAP-pipeline live: HTTPS-bestemming gebumpt (`O=SASE PoC`), DPI op de payload, geblokkeerd met leesbare reden (`YARA.DLP_Confidential_Label`). Bewijst end-to-end: TLS-decryptie → DPI → blokkade. |
| T-ME2 | QUIC-blokkade forceert TCP/443-fallback → Squid bumpt → ClamAV/DLP inspecteren de volledige payload. |

### Algemeen: Troubleshooting

**Rubric-criterium:** diepgaande analyse.

| Test | Wat het bewijst |
|---|---|
| T-TP1-T-TP4 | Gelaagde diagnostiek: elke laag van de stack individueel gevalideerd (`tcpdump`, `pfctl -vvs nat`, `netstat -an`, conntrack-analyse). Root cause geïsoleerd via twee gelijktijdige tcpdumps (ens3 versus wt0). Foute diagnose (01.9) gecorrigeerd op basis van empirisch bewijs. |

### Algemeen: Architectuur

**Rubric-criterium:** professioneel.

| Bewijs | Wat het bewijst |
|---|---|
| Vijf-optie-analyse | Systematische evaluatie van architectuuropties met voor/tegen per optie. |
| Optie C implementatie + verwerping | Hypothese getest, weerlegd en gecorrigeerd: architectuurbeslissing op basis van bewijs, niet aanname. |
| Managed-device pivot | Architecturele herstructurering op basis van veranderde aannames (Intune-enrollment maakt netwerkinterceptie niet-vereist voor beheerde devices). |

### Algemeen: Samenwerking

**Rubric-criterium:** proactieve samenwerking, kennisdeling en gezamenlijke probleemoplossing.

| Bewijs | Wat het bewijst |
|---|---|
| Finding + decision doc van het teamlid | Probleemidentificatie en eerste architectuurvoorstel door een teamlid. |
| TransProxy_Verslag02 | Diagnostiek bouwt voort op de implementatie van het teamlid; corrigeert zijn foute diagnose (01.9) maar valideert zijn architectuurkeuze (Optie B). |
| Revert `.11`-omgeving | Na diagnostiek teruggezet naar de werkende staat (masquerade ON): geen regressie voor het team. |

---

## Testdetails

### T-TP1: source-IP bewaard na masquerade OFF

**Doel:** bewijzen dat het overlay-IP intact blijft na het verwijderen van NetBird-masquerade.  
**Voorwaarde:** NetBird exit node hermaakt als Network Route (`0.0.0.0/0`, masquerade OFF).  
**Commando:** `tcpdump -i ens3 -n 'tcp port 443'` op linuxpop01 tijdens `curl` vanaf mobile01.  
**Acceptatiecriterium:** `src=100.86.x.x` (overlay-IP), niet `192.168.122.17` (linuxpop01 LAN-IP).  
**Gemeten resultaat:** ✅ Source-IP behouden. Bevestigt dat bevinding 01.9 ("Linux herschrijft source IP") fout was: de masquerade zat in nftables `netbird-rt-postrouting`.

### T-TP2: pf rdr matcht overlay-verkeer op vtnet0

**Doel:** bevestigen dat OPNsense pf rdr het overlay-verkeer correct onderschept na aankomst op vtnet0.  
**Commando:** `pfctl -vvs nat` op pop01.  
**Acceptatiecriterium:** state creations > 0 op de rdr-regels voor `100.64.0.0/10`.  
**Gemeten resultaat:** ✅ 536 state creations op de HTTPS-regel. pf rdr functioneert correct: de blocker zit niet hier.

### T-TP3: `SYN_RCVD`-hang, return-pad asymmetrisch

**Doel:** bewijzen dat het return-pad structureel gebroken is bij source-IP-behoud met interceptie achter een tweede stateful router.  
**Commando's (gelijktijdig):**

- `netstat -an | grep SYN_RCVD` op pop01
- `tcpdump -i ens3 -n 'src 192.168.122.11'` op linuxpop01
- `tcpdump -i wt0 -n` op linuxpop01

**Acceptatiecriterium:** SYN-ACK arriveert op ens3 maar verschijnt niet op wt0 (conntrack-mismatch → DROP).  
**Gemeten resultaat:** ✅ Precies zoals voorspeld. Dit is het beslissende bewijs dat Optie C architectureel niet kan werken met source-IP-behoud. Het is geen configuratiefout, maar een fundamentele eigenschap van de netfilter conntrack-state-machine.

### T-TP4: masquerade ON maskeert het probleem

**Doel:** verklaren waarom de implementatie met masquerade ON wél werkte.  
**Analyse:** met masquerade ON was de source `192.168.122.17`, het lokale IP van linuxpop01. Squid's antwoord ging naar een lokaal IP → lokale aflevering → linuxpop01's conntrack de-NAT'te het terug naar de client. Geen forwarding-pad, dus geen conntrack-mismatch.  
**Gemeten resultaat:** ✅ Verklaart het mechanisme. De prijs: source-IP verloren → Identity Bridge onbruikbaar.

### T-TP5: TPROXY source-IP-behoud (Optie D)

**Doel:** bewijzen dat Squid op linuxpop01 via TPROXY het echte client-overlay-IP ziet.  
**Commando:** `tail -f /var/log/squid/access.log` op linuxpop01 tijdens browse vanaf mobile01.  
**Afgeleid resultaat:** `src = 100.86.x.x` (niet het linuxpop01-IP).  
**Status:** ✖ Niet uitgevoerd. linuxpop01 bestaat niet op de sandbox; Sessie 9 is gedeprioriteerd omdat managed-device enforcement (V41) het primaire pad dekt. De voorgestelde oplossing: Squid met TPROXY op linuxpop01 intercepteert in `iptables mangle PREROUTING` op de `wt0`-interface, vóór de forwarding-beslissing. De `IP_TRANSPARENT`-socketoptie behoudt per definitie zowel src als dst; dit is een gedocumenteerde kernel-eigenschap, geen hypothese. Zie [Transparante proxy: TPROXY op linuxpop01](../components/transparent-proxy.md) voor de volledige architectuur en implementatiespecificatie.

### T-TP6: symmetrisch return-pad (Optie D)

**Doel:** bewijzen dat het return-pad nu symmetrisch is met TPROXY (het probleem dat Optie C nekte).  
**Commando:** `tcpdump -i wt0 -n` op linuxpop01 tijdens `curl` vanaf mobile01.  
**Afgeleid resultaat:** SYN-ACK verschijnt terug op wt0 (geen `SYN_RCVD`-hang).  
**Status:** ✖ Niet uitgevoerd (zelfde reden als T-TP5). Dit resultaat volgt logisch uit de architectuur: Squid op linuxpop01 is zowel interceptiepunt als connectie-eindpunt; het pakket wordt in `mangle PREROUTING` naar de lokale socket geleid en verlaat de forwarding-pijplijn nooit. De asymmetrische return-pad-mismatch die T-TP3 bewees (conntrack op een tussenliggende router dropt de SYN-ACK) kan hier structureel niet optreden. Zie [De diagnostiek: waarom Optie C faalt](../components/transparent-proxy.md#de-diagnostiek-waarom-optie-c-faalt) voor het contrast.

### T-ME1: Intune proxy-CSP, bump + overlay-`%SRC` in access.log

**Doel:** bewijzen dat de Intune-pushed proxy systeembreed werkt en het overlay-IP doorgeeft.  
**Commando's:**

- Op mobile01: browse naar `google.com`, controleer de certificaat-issuer.
- Op pop01: `tail -f /var/log/squid/access.log`.

**Acceptatiecriterium:** issuer = `O=SASE PoC` (bump actief); access.log toont `100.70.178.122` (overlay-IP, geen LAN-IP).  
**Gemeten resultaat:** ✅ Gevalideerd (V41 41.11). Microsoft-domeinen correct gespliced (no-bump).

### T-ME2: Intune firewall-CSP, QUIC→TCP-fallback + SSL Bump

**Doel:** bewijzen dat de Intune firewall-blokkade van QUIC (UDP/443) browsers naar TCP/443 forceert, waar Squid het inspecteert.  
**Commando:** browse naar `google.com` (Google is een QUIC-property); controleer de certificaat-issuer.  
**Acceptatiecriterium:** certificaat-issuer = `O=SASE PoC`; de browser viel terug op TCP/443 en Squid bumpte het.  
**Gemeten resultaat:** ✅ Gevalideerd (V41 41.19/41.20). De bump bewijst dat QUIC geblokkeerd werd.

### T-ME3: Intune firewall-CSP, tunnel + web + MDM intact

**Doel:** bewijzen dat de firewall-blokkades (QUIC/DoT) geen legitiem verkeer breken.  
**Controles:**

1. `netbird status` → Connected (de tunnel overleeft; de UDP/443-block raakt de 51820-tunnel niet).
2. `google.com` laadt in de browser (web via de proxy intact).
3. Intune sync successful (MDM-check-in via het WinHTTP-SYSTEM-pad open).

**Acceptatiecriterium:** alle drie groen.  
**Gemeten resultaat:** ✅ Gevalideerd (V41 41.19). Drie B3-criteria bewezen: "firewall breekt geen legitiem verkeer".

### T-ME4: DLP-blokkade end-to-end via afgedwongen proxy

**Doel:** bewijzen dat de volledige DPI-keten werkt via de Intune-afgedwongen proxy.  
**Gemeten resultaat (en passant bij T-ME2):** de DLP-ICAP-pipeline logde live een geblokkeerde transfer:

```
malware: YARA.DLP_Confidential_Label.UNOFFICIAL
HTTP location: https://docs.cloud.google.com/...
```

Dit bewijst end-to-end: HTTPS-bestemming gebumpt (anders kon ICAP de payload niet zien) → DPI + TLS-decryptie → geblokkeerd met leesbare reden.  
**Beoordeling:** ✅ Gevalideerd (V41 41.20). Raakt rubric SWG Malware-inspectie ("DPI + TLS-decryptie + malware-detectie") en Algemeen Transparantie ("uitleg waarom geblokkeerd").

---

## Gerelateerd

- [Transparante proxy: TPROXY op linuxpop01](../components/transparent-proxy.md)
- [Acceptatietests (F1-F15)](acceptance-tests.nl.md)
- [Aanvals- & bypass-scenario's](attack-scenarios.nl.md)

---
title: "Transparante proxy: TPROXY op linuxpop01"
tags: [tproxy, squid, proxy, netbird, enforcement, sase, identity-bridge, intune]
---

# Transparante proxy: TPROXY op linuxpop01

**Rol:** Defense-in-depth enforcement voor niet-beheerde (BYOD) devices. Transparante interceptie van al het HTTP/HTTPS-verkeer dat via de NetBird-overlay binnenkomt, ongeacht de clientconfiguratie.  
**Status:** ✅ PoC-validated op de parallelle stack (diagnostiek); Optie D ontworpen, sandbox-integratie pending. De architectuurkeuze is gevalideerd op de `.11`-omgeving, maar nog niet in het sandbox-geheel geïntegreerd: Sessie 9 is gedeprioriteerd omdat managed-device enforcement via Intune het primaire pad dekt (V41).  
**Node:** linuxpop01 (Ubuntu 24.04). Op de parallelle stack is dit `192.168.122.17`; de voorgestelde sandbox-node krijgt `192.168.122.43` en is nog niet aangemaakt.

---

## Probleemstelling

De sandbox gebruikt een [expliciete proxy via WPAD/PAC](../decisions/wpad-vs-transparent-proxy.md): clients sturen hun HTTP/HTTPS-verkeer naar Squid op `100.70.154.79:3128` omdat het PAC-bestand dat voorschrijft. Dit is een client-side compliance model. De inspectie werkt alleen als de applicatie de systeemproxyinstelling respecteert.

Het probleem zit bij applicaties met een eigen netwerkstack (Spotify, Teams system services, game launchers, PowerShell-scripts). Die negeren de WinINET-proxy en sturen hun verkeer rechtstreeks via de NetBird-tunnel. Dat verkeer exit-t ongeïnspecteerd via de gemasqueradede tunnel: geen Squid-log, geen DLP, geen URL-filter, geen Identity Bridge-trace.

Dit raakt de rubric op twee punten:

| Rubric-criterium | Impact |
|---|---|
| **SWG, Traffic routing** | "Volledig geforceerd via SWG" vereist dat *al* het verkeer door de inspectiestack passeert, niet alleen browserverkeer. |
| **SWG, Identity-based access** | De Identity Bridge (`external_acl_type` met `%SRC`) werkt alleen voor verkeer dat Squid passeert. Non-proxy-verkeer is onzichtbaar. |

De fundamentele vraag: hoe dwing je inspectie af voor verkeer dat de proxy negeert?

---

## Onderzochte opties

Het onderzoek analyseerde vijf architectuuropties. Optie B was operationeel op de `.11`-omgeving (de werkende implementatie met masquerade ON). Optie C is grondig geïmplementeerd en in meerdere configuraties getest op dezelfde omgeving. De diagnostiek die daaruit volgde is de kern van dit document. De volledige analyse staat in Addendum F v2 (vijf-optie-analyse) en Addendum F v3 (herziening na diagnostiek).

### Optie A: TPROXY + cache_peer (dubbele Squid)

linuxpop01 intercepteert via TPROXY en stuurt door naar pop01 Squid via `cache_peer`.

**Verworpen.** SSL Bump + cache_peer is fundamenteel incompatibel. Twee scenario's, beide gebroken: (1) beide Squids bumpen, dus dubbele MITM en TLS breekt; (2) alleen pop01 bumpt, dus linuxpop01 kan de CONNECT-tunnel niet inspecteren. Architectureel niet op te lossen zonder de SSL Bump-keten te doorbreken.

### Optie B: TPROXY standalone op linuxpop01

linuxpop01 draait een eigen Squid met TPROXY (`IP_TRANSPARENT`), SSL Bump en ICAP. Squid op linuxpop01 is zowel interceptiepunt als connectie-eindpunt.

**Architectureel correct**, de oorspronkelijke keuze. De prijs: een tweede Squid-instantie met eigen SSL Bump- en ICAP-configuratie. In v2 werd deze optie gepasseerd ten gunste van Optie C om één Squid te behouden. In v3 bleek Optie C onhoudbaar en is Optie B (hernoemd naar Optie D met verfijningen) alsnog de gekozen architectuur.

### Optie C: routing-only gateway + pf rdr op pop01

linuxpop01 als lichtgewicht NAT-gateway (alleen `ip_forward`, geen Squid), met transparante `pf rdr`-interceptie op pop01 (OPNsense).

**Verworpen na implementatie en diagnostiek.** Optie C is grondig geïmplementeerd en getest op de `.11/.17`-omgeving. De diagnostiek leverde een sluitend causaal model op voor het falen. Zie [De diagnostiek: waarom Optie C faalt](#de-diagnostiek-waarom-optie-c-faalt).

### ~~mgmt01 als exit node~~ (verworpen)

**Verworpen op architectuurprincipe.** Vermengt het control plane (NetBird management, Zitadel, Caddy) met het data plane (exit node). Een schending van het Zero Trust blast radius-principe.

### ~~NetBird-route zonder exit node~~ (verworpen)

Variant van Optie C. Zelfde routing-pad, zelfde return-pad-probleem.

---

## De diagnostiek: waarom Optie C faalt

*Bron: TransProxy_Verslag02 (30-31 mei 2026), de volledige diagnostische sessie op de parallelle stack.*

De diagnostiek begon als review van de werkende implementatie (masquerade ON) en escaleerde naar een volledige ontrafeling van het return-pad-probleem. De kernbevinding: transparante proxy-interceptie op een node achter een tweede stateful router kan het verkeer niet correct retourneren zodra het source-IP behouden wordt.

### Stap 1: source-IP-verlies opsporen

Bevinding 01.9 ("Linux herschrijft source IP bij forwarding") bleek fout gediagnosticeerd. Linux-forwarding herschrijft nooit source-IP's zonder expliciete SNAT of MASQUERADE. De werkelijke oorzaak: NetBird's "Add Exit Node"-functie forceert masquerade ON in een nftables-chain (`netbird-rt-postrouting`), zonder UI-toggle. De NetBird-documentatie bevestigt dit letterlijk.

**Oplossing:** exit node verwijderen en hermaken als reguliere Network Route (`0.0.0.0/0`, masquerade OFF).

### Stap 2: masquerade OFF, pf rdr matcht

Na het verwijderen van de masquerade komt het overlay-IP (`100.86.x.x`) intact aan op pop01 vtnet0. De `pf rdr`-regels matchen correct:

```
@6 rdr on vtnet0 inet proto tcp from 100.64.0.0/10 ... port = https -> 192.168.122.11 port 3130
   [ Packets: 10275  State Creations: 536 ]
```

### Stap 3: Squid ontvangt de SYN, maar de client krijgt niets terug

De TCP-state op pop01 tijdens een verbinding:

```
tcp4  192.168.122.11:3129  100.86.233.213:57989  SYN_RCVD
```

`SYN_RCVD` is het beslissende bewijs: Squid ontving de SYN, de kernel stuurde een SYN-ACK, maar de ACK van de client komt nooit. Twee gelijktijdige tcpdumps op linuxpop01 bevestigden het: de SYN-ACK arriveert onvertaald op ens3 (`src=192.168.122.11:3129`), maar verschijnt nooit op wt0. Conntrack dropt hem.

### Stap 4: het causale model

```
Squid antwoordt vanaf 192.168.122.11:3129
  → pf reverse-translateert NIET (lokaal getermineerde verbinding)
  → SYN-ACK vertrekt onvertaald: 192.168.122.11:3129 → 100.86.233.213
  → Routeert via statische route naar linuxpop01
  → Op linuxpop01: conntrack verwacht antwoord van origin:80, krijgt 192.168.122.11:3129
  → INVALID → FORWARD RELATED,ESTABLISHED → DROP
  → Client krijgt niets → SYN_RCVD hang → timeout
```

Dit is een architecturele mismatch, geen configuratiefout. De interceptie zit op de verkeerde node: een pf rdr achter een tweede stateful router (linuxpop01's conntrack) kan niet symmetrisch retourneren. Dit is precies de reden waarom Linux TPROXY bestaat.

### Waarom masquerade ON het maskeerde

Met masquerade ON was de source `192.168.122.17` (linuxpop01's eigen IP). Squid's antwoord ging naar een lokaal IP, dus lokale aflevering en geen forwarding. linuxpop01's masquerade-conntrack de-NAT'te het terug naar de client. Het werkte niet *dóór* de masquerade, maar doordat de source toevallig lokaal was. De prijs: source-IP verloren, en daarmee de Identity Bridge onbruikbaar.

---

## Gekozen architectuur: Optie D (TPROXY op linuxpop01)

*Bron: Addendum F v3, TPROXY Herziening (31 mei 2026).*

```
mobile01
  └─[WireGuard]──► linuxpop01 (wt0)
                     │
                     │ iptables mangle PREROUTING
                     │   TPROXY: TCP/80  → lokale Squid :3129
                     │   TPROXY: TCP/443 → lokale Squid :3130
                     ▼
                   Squid op linuxpop01
                     │  ziet ORIGINELE src (100.86.x.x) + dst
                     │  SSL Bump (zelfde SASE-PoC-CA)
                     │  ICAP RESPMOD → ClamAV  (pop01:1344)
                     │  ICAP REQMOD  → Python DLP (mgmt01:1345)
                     │
                     └──► ens3 ──► pop01 ──► internet
```

> **Let op (Optie D is ontworpen, niet getest).** Dit ontwerp is architectureel onderbouwd in Addendum F v3, maar nog niet empirisch gevalideerd. De diagnostiek (Optie C faalt) is wél getest op de parallelle stack; de TPROXY-tests 5 en 10 (source-IP-behoud en symmetrisch return-pad) zijn niet uitgevoerd omdat Sessie 9 is gedeprioriteerd. Zie [Testblok: transparante proxy en enforcement](../testing/transparent-proxy-tests.md) (T-TP5/T-TP6).

### Waarom dit symmetrisch is

Squid op linuxpop01 is zowel het interceptiepunt als het connectie-eindpunt. De interceptie gebeurt in `mangle PREROUTING`, vóór de forwarding-beslissing. Het pakket wordt naar de lokale socket geleid en verlaat de forwarding-pijplijn nooit. Er is geen tweede stateful router tussen client en proxy die de flow-symmetrie kan breken.

### Source-IP-behoud via IP_TRANSPARENT

TPROXY markeert het pakket en de kernel routeert het naar een lokale socket met de `IP_TRANSPARENT`-socketoptie. Squid's tproxy-listener ontvangt de verbinding met zowel src als dst intact. Daarmee logt Squid het echte client-overlay-IP, de voorwaarde voor de [Identity Bridge](identity-bridge.md) (`external_acl_type` met `%SRC` → NetBird overlay-IP → Entra ID-gebruiker).

### Twee complementaire Squids

| | pop01 Squid | linuxpop01 Squid |
|---|---|---|
| Modus | expliciet (WPAD/PAC, `:3128`) | transparant (TPROXY, `:3129`/`:3130`) |
| Doel | clients die de proxy respecteren | enforcement voor al het overige verkeer |
| SSL Bump | ja (zelfde CA) | ja (zelfde CA) |
| ICAP | ClamAV lokaal + DLP mgmt01 | ClamAV pop01 + DLP mgmt01 (over netwerk) |
| Identity Bridge | optioneel | **primair** (ziet echte src-IP) |

WPAD/PAC en TPROXY zijn complementair, niet concurrerend: WPAD is het configuratiemechanisme voor compliant verkeer, TPROXY is de enforcement-boundary voor al het overige.

---

## De managed-device pivot

*Bron: SD-WAN Nulmeting v1.0 §2.1 (3 juni 2026); V41 (Intune-enforcement sessie).*

De non-browser-apps-gap was het enige dat transparante interceptie vereiste, en dan nog alleen voor **niet-beheerde** clients. Met mobile01 nu Entra-joined + Intune-enrolled (V40) en de ingelogde user géén lokale admin, is endpoint-enforcement via MDM beschikbaar zonder dat de user het kan terugdraaien. Dit is exact het patroon van Zscaler ZCC en de Netskope-client.

V41 leverde het bewijs dat managed-device enforcement werkt:

| Intune-profiel | Wat het afdwingt | Bewijs |
|---|---|---|
| `2ITCSC1A-SASE-PoC-CA-Root` | SASE-PoC-CA in Computer Root store | Vervangt de handmatige `certutil`-stap |
| `2ITCSC1A-SASE-Proxy-PAC` | Systeembrede proxy via OMA-URI | Overlay-`%SRC` in pop01 access.log (B1) |
| `2ITCSC1A-SASE-FW-BypassBlock` | QUIC UDP/443 + DoT TCP/853 geblokkeerd | QUIC→TCP-fallback + bump bewezen (B3) |
| `2ITCSC1A-Route-Remediation` | Teams Optimize via fysieke gateway | `Find-NetRoute` → fysieke gw (B2) |

**Gevolg:** TPROXY/linuxpop01 is geen prerequisite meer voor de SD-WAN-pijler en evenmin voor beheerde devices. linuxpop01 keert terug naar zijn oorspronkelijke rol: defense-in-depth voor niet-beheerde devices. Sessie 9 (TPROXY-implementatie) is gedeprioriteerd ten gunste van de rubric-dragende sessies en niet meer uitgevoerd binnen de projectperiode. De architectuur is volledig uitgewerkt en gevalideerd op de `.11`-omgeving; de implementatiespecificatie (Addendum F v3) ligt klaar voor eventuele toekomstige uitvoering. Zie [Beslissing: scope beheerde Windows-apparaten](../decisions/managed-devices-scope.md).

---

## Proxy Bypass-mitigaties

*Bron: `SASE_PoC_Proxy_Bypass_Mitigaties.md` (mei 2026).*

Het Proxy Bypass-document inventariseert vier resterende omzeilingsmogelijkheden na implementatie van transparante interceptie:

| Gap | Aanval | Mitigatie | Status |
|---|---|---|---|
| **1. QUIC** | HTTP/3 over UDP/443 passeert de TCP-interceptie | `iptables -A FORWARD -i wt0 -p udp --dport 443 -j DROP` op linuxpop01; de browser valt terug op TCP/443 | Op de sandbox afgedekt door de Intune firewall-CSP (V41) |
| **2. Non-standard poorten** | SSH-tunnel op 22, SOCKS5 op 1080, enzovoort | Deny-all FORWARD + expliciete whitelist (80, 443, 53, 123, ICMP) | Architectureel uitgewerkt; productie-aanpak is full default-deny |
| **3. DoH/DoT** | DNS over HTTPS/TLS omzeilt Unbound RPZ | DoT: TCP/853 geblokkeerd (deny-all of expliciete regel); DoH: Squid-ACL blokkeert bekende resolvers op SNI | Op de sandbox afgedekt door de Intune firewall-CSP (V41) |
| **4. VPN-in-VPN** | Tweede WireGuard/OpenVPN-client op het device | Drie-gate circulaire afhankelijkheid: CA compliance → NetBird auth → Intune firewall rules. Architecturele grens: verdedigd, niet opgelost. | Gedocumenteerd als bekende beperking |

Gaps 1-3 verhuizen van pf-regels op pop01 naar iptables FORWARD op linuxpop01 zodra TPROXY geïmplementeerd wordt. Dat lost en passant bevinding 01.11 op: de floating-rule/`quick`-interactie die op OPNsense de pf rdr brak, bestaat niet in iptables.

---

## Rubric-relevantie

De transparante proxy en het bijbehorende onderzoek raken meerdere rubric-criteria, ook zonder live implementatie op de sandbox:

| Rubric-criterium | Bijdrage |
|---|---|
| **SWG, Traffic routing** | Het onderzoek toont architecturaal aan hoe "volledig geforceerd via SWG" bereikt wordt voor non-browser-verkeer. De managed-device-variant (Intune) is live bewezen (V41). |
| **SWG, Identity-based access** | TPROXY behoudt het client-overlay-IP (`%SRC`), de voorwaarde voor de Identity Bridge (overlay-IP → Entra ID-gebruiker → persona-ACL). Zonder source-IP-behoud is identity-based filtering onmogelijk voor transparant geïntercepteerd verkeer. Dit was de kernmotivatie voor het hele onderzoek. |
| **SWG, Malware-inspectie** | Optie D behoudt de volledige ICAP-pijplijn (ClamAV + DLP) voor transparant geïntercepteerd verkeer. |
| **Algemeen, Troubleshooting** | De `SYN_RCVD`-diagnostiek (TransProxy_Verslag02) is een diepgaande analyse van een asymmetrisch return-pad, precies het type root cause-analyse dat "Diepgaande analyse" vereist. |
| **Algemeen, Architectuur** | Vijf opties systematisch geanalyseerd, drie geïmplementeerd en getest, foute diagnose gecorrigeerd, architectuurbeslissing herzien op basis van bewijs. |
| **Algemeen, Keuze open-source tools** | Afweging pf (FreeBSD) versus iptables/TPROXY (Linux) op basis van kernel-specifieke beperkingen, niet op voorkeur. |
| **Algemeen, Samenwerking** | De finding en het decision doc van het teamlid vormden het startpunt; de diagnostiek bouwde voort op zijn implementatie; zijn Optie B werd architecturaal gevalideerd. |

---

## Bekende problemen / valkuilen

**`pf rdr` werkt niet op `wt0`.** Dit is de oorspronkelijke beperking (V17, plugins#3857) die het hele onderzoek in gang zette. WireGuard-forwarded-verkeer wordt na decapsulatie direct in de kernel routing stack geïnjecteerd en verschijnt als egress, niet als ingress. Zie [Bevinding: wt0 pf rdr-beperking](../findings/wt0-pf-rdr-limitation.md).

**"Add Exit Node" forceert masquerade ON.** NetBird's exit-node-functie past masquerade toe in nftables (`netbird-rt-postrouting`) zonder UI-toggle. Gebruik in plaats daarvan een Network Route (`Add Route`, range `0.0.0.0/0`, masquerade OFF) om het source-IP te behouden.

**rdr naar 127.0.0.1 faalt op FreeBSD.** Squid's NAT-lookup kan de oorspronkelijke bestemming niet ophalen bij een rdr naar loopback voor geforward verkeer. De error: `NAT lookup failed to locate original IPs`. Redirect naar het WAN-IP lost dit op, maar introduceert het return-pad-probleem.

**iptables FORWARD-volgorde.** libvirt (op de GNS3-host) injecteert een `REJECT`-regel in de FORWARD-chain. Gebruik altijd `-I FORWARD 1` in plaats van `-A FORWARD`. Zie [Bevinding: iptables FORWARD-volgorde](../findings/iptables-forward-ordering.md).

**TPROXY routing-persistentie.** `ip rule add fwmark 1 lookup 100` en `ip route add local 0.0.0.0/0 dev lo table 100` overleven geen reboot. Vastleggen via een systemd-unit of een netplan-hook.

---

## Gerelateerd

- [Squid: expliciete proxy](squid.md): de complementaire proxymodus op pop01
- [Identity Bridge](identity-bridge.md): vereist het echte client-overlay-IP dat TPROXY behoudt
- [Beslissing: WPAD/PAC vs. transparante proxy](../decisions/wpad-vs-transparent-proxy.md): de oorspronkelijke keuze (V17/V19)
- [Beslissing: scope beheerde Windows-apparaten](../decisions/managed-devices-scope.md): de pivot naar endpoint-enforcement
- [Bevinding: wt0 pf rdr-beperking](../findings/wt0-pf-rdr-limitation.md): de trigger voor het onderzoek
- [Bevinding: iptables FORWARD-volgorde](../findings/iptables-forward-ordering.md): relevant voor de TPROXY iptables-configuratie
- [Concept: SSL Bump](../concepts/ssl-bump.md): gedeelde CA-keten tussen beide Squids
- [Testblok: transparante proxy en enforcement](../testing/transparent-proxy-tests.md): de diagnostische tests en de managed-device-variant
- [Runbook 03: Proxy & WPAD](../runbooks/03-proxy-wpad.md): operationele instructies voor de expliciete proxy

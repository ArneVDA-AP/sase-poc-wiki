---
title: "NetBird + Zitadel + Entra ID: ZTNA-overlay"
tags: [netbird, zero-trust, sase, network, opnsense, docker]
---

# NetBird + Zitadel + Entra ID: ZTNA-overlay

**Rol:** ZTNA-transportlaag die een WireGuard-mesh-overlay creëert die managed Windows-clients toegang geeft tot pop01 (proxy, DNS) en DC-LAN-resources, ongeacht hun fysieke netwerklocatie. Elke toegangsbeslissing is identiteitsgebaseerd; lateral movement tussen resources vereist expliciet ACL policy.  
**Status:** ✅ Volledig operationeel (snapshot `Fase2-ZTNA-Complete`)  
**Configuratielocatie:** mgmt01 Docker Compose (`~/docker-compose.yml`), NetBird Dashboard (`https://netbird.sandbox.local`)

---

## Werking in deze stack

Zonder NetBird heeft mobile01 geen netwerkpad naar pop01. De gehele SWG-pijplijn (Squid, ClamAV, DLP, Unbound RPZ) is afhankelijk van mobile01 dat de overlay-IP `100.70.154.79:3128` van pop01 bereikt. NetBird creëert de versleutelde tunnel die dit mogelijk maakt vanuit elk netwerk (thuis, mobiel, school) zonder enige service publiek bloot te stellen.

**De IdP-keten:**

```
Gebruiker → NetBird Dashboard → Zitadel (OIDC-uitgever, zelf-gehost op mgmt01)
                                        ↓
                               Entra ID (externe IdP via OIDC-federatie)
                                        ↓
                                Microsoft-inlogpagina (aplab.be-tenant)
```

Het quickstart-script van NetBird installeert Zitadel als primaire OIDC-uitgever. Entra ID is verbonden als externe IdP *aan Zitadel*, niet rechtstreeks aan NetBird. Dit is een architecturele realiteit van het quickstart-script, geen beperking. Zitadel voegt een centrale gebruikersbeheerlaag toe met rollen en groepen die onafhankelijk zijn van Entra ID-configuratie.

**Entra ID Conditional Access** evalueert bij het Entra ID `/authorize`-endpoint, gericht op de NetBird-app-registratie `2ITCSC1A-Netbird-Sandbox` (`11803ee8-eb15-462c-a286-5415c17a29c6`). Of de OIDC-omleiding afkomstig is van Zitadel of rechtstreeks van een NetBird-client is irrelevant: de gebruiker authenticeert zich rechtstreeks bij Entra ID en CA wordt geactiveerd voor die app-registratie. Zie [Beslissing: CA-posture hybride](../decisions/ca-posture-hybrid.md).

**WireGuard-mesh vs. per-app-tunnels:** NetBird creëert een volledige WireGuard-mesh waarbij ACL policy bepaalt welke peers kunnen communiceren. Dit is architectureel equivalent aan per-app Zero Trust-tunnels (Zscaler ZPA): een peer met toegang tot dc01 kan mgmt01 niet bereiken zonder een aparte ACL policy (resources zijn niet zichtbaar, niet alleen ontoegankelijk).

---

## Configuratie

### Implementatie: Zitadel-quickstart

```bash
curl -sSLO https://github.com/netbirdio/netbird/releases/latest/download/getting-started-with-zitadel.sh
chmod +x getting-started-with-zitadel.sh
sudo ./getting-started-with-zitadel.sh
# Bij prompt: domein = netbird.sandbox.local, IdP = None (gebruik ingebouwde Zitadel)
# Script MISLUKT bij Let's Encrypt — dit is verwacht. Stop met Ctrl+C.
```

Het script mislukt omdat Let's Encrypt geen certificaten kan uitreiken voor privé-hostnamen. Patch na het aanmaken van de Docker Compose-stack handmatig het Caddyfile:

```caddyfile
netbird.sandbox.local {
    tls internal    # ← voeg deze regel toe
    # ... bestaande reverse_proxy-configuratie
}
```

Voeg een Docker-netwerkalias toe zodat interne containers de hostnaam kunnen resolven:

```yaml
services:
  caddy:
    networks:
      netbird:
        aliases:
          - netbird.sandbox.local
```

```bash
sudo docker compose down && sudo docker compose up -d
curl -k https://netbird.sandbox.local/zitadel/debug/ready
# Verwacht: "ok" of HTTP 200
```

Controleer of `management.json` verwijst naar `netbird.sandbox.local`, niet naar bare IP's.

### Hosts-vermeldingen

Elke machine die communiceert met NetBird moet de hostnaam kunnen resolven:

| Machine | Methode |
|---------|---------|
| mgmt01 | `/etc/hosts`: `192.168.122.23 netbird.sandbox.local` |
| pop01 (OPNsense) | Console optie 8: `echo "192.168.122.23 netbird.sandbox.local" >> /etc/hosts` |
| mobile01 (Windows) | `C:\Windows\System32\drivers\etc\hosts` (als beheerder) |

De GNS3-host draait nginx SNI stream passthrough: `netbird.sandbox.local:443` → `192.168.122.23:443`.

### Entra ID externe IdP

Via Zitadel-console op `https://netbird.sandbox.local/zitadel/ui/console`:

```
Settings → Identity Providers → Add → Microsoft Azure AD
Client ID:     11803ee8-eb15-462c-a286-5415c17a29c6
Tenant ID:     23e9bcdc-5cb9-4867-9310-76cc0b462ddc
Client Secret: <uit Entra ID-app-registratie>
```

Voeg de door Zitadel gegenereerde omleidings-URI toe aan de Entra ID-app-registratie → Authenticatie.

**Gebruikersgoedkeuring:** Nieuwe gebruikers die via Entra ID authenticeren, verschijnen als "in behandeling" in NetBird Dashboard → Gebruikers. Een beheerder moet ze handmatig goedkeuren; dit is een bewuste beveiligingsfunctie van de Zitadel-quickstart, geen bug.

### Groepen

| Groep | Leden | Doel |
|-------|-------|------|
| `Core-Services` | pop01, mgmt01 | Infrastructuurpeers (proxy, DNS, WPAD, ICAP) |
| `Studenten` | student-persona-peers | Studentclients (JWT auto-groep) |
| `Docenten` | docent-persona-peers | Docentclients (JWT auto-groep) |
| `Admins` | admin-persona-peers | Administratieve gebruikers (JWT auto-groep) |
| `All` | elke peer | NetBird ingebouwde auto-groep |

Alleen `Core-Services` wordt handmatig aangemaakt. De persona-groepen (`Studenten`/`Docenten`/`Admins`) worden **niet** in het dashboard aangemaakt: een JWT-groep materialiseert pas als NetBird-groep wanneer een peer met die `groups`-claim voor het eerst verbindt; enrollment *is* de groepssynchronisatie (geen aparte stap).

> **Groepsmigratie (V34, mei 2026).** De oorspronkelijke Fase-2-build gebruikte `SASE-Admins`, `SASE-MobileUsers`, `SASE-Services` en `SASE-InternalResources`. Een diagnose toonde dat **alle** NetBird ACL-connectiviteit op de `SASE-*`-groepen hing, terwijl de persona-groepen **nul** policies droegen. Quarantaine-per-groep (een peer uit zijn persona-groep verwijderen) zou dus een no-op zijn geweest. De migratie verplaatste alle connectiviteit naar het persona-model + `Core-Services` en verwijderde elke `SASE-*`-groep en -policy. Het persona-model hieronder is de huidige staat; het `SASE-*`-model is verouderd.

**Policy creation order (deny-by-default):** De default all-to-all policy is verwijderd, dus peers zonder policy hebben geen connectiviteit. Maak de vervangende policy aan *vóór* het verwijderen van de default, om te voorkomen dat alle peer-communicatie wegvalt.

### ACL policy

**`Personas-to-Core-Services`:** Sources: `Studenten`, `Docenten`, `Admins` → Destination: `Core-Services`, Protocol: **alleen TCP 3128**.

Deze ene policy is de volledige allow-list onder deny-by-default. Het laat elke persona-peer de pop01 Squid-proxy (`100.70.154.79:3128`) bereiken; al het webverkeer loopt vervolgens door de SWG-pijplijn, waar de differentiatie per persona in Squid gebeurt (niet in NetBird-ACL's). WPAD-poorten (`:80`/`:443` op mgmt01) zijn **bewust weggelaten**: het PAC-bestand is leeg en clients gebruiken een handmatige proxy-instelling, dus WPAD-bereikbaarheid is niet nodig (V34.13). Quarantaine werkt omdat het verwijderen van een peer uit zijn persona-groep zijn enige pad naar `Core-Services` wegneemt.

### Exit node en DC-LAN-routering

**Exit node (internetverkeer):** via Netwerkroutes:
```
Netwerk: 0.0.0.0/0, Routing Peer: pop01, Distribution Groups: Studenten, Docenten, Admins
```

Het exit node blijft bestaan zodat persona-groepverkeer (Studenten/Docenten/Admins) via pop01 naar buiten gaat. Het exit node van NetBird is **alles-of-niets**: het kan geen destination ranges selectief uitsluiten (issues #2493 / #3523). Om Microsoft 365 "Optimize"-bereiken van de tunnel te houden, pusht een Intune-Remediation (`2ITCSC1A-Route-Remediation`) die routes rechtstreeks op de client (een client-side split-tunnel die het ontbrekende per-route-uitsluiten compenseert).

**DC-LAN (10.0.0.0/8) over de overlay (uitgesteld).** Het oorspronkelijke `Internal-DC`-netwerk (ACL-bewust) en zijn `Datacenter-Access`-policy zijn **verwijderd in de V34-migratie**; er is geen actief DC-LAN-over-overlay-pad in de huidige sandbox. Het opnieuw introduceren van datacenterbereikbaarheid is uitgesteld tot de geplande Cosmos-app-gateway-sessie. De ontwerpredenering hieronder blijft behouden voor wanneer dat werk wordt hervat.

> **Netwerken vs. Netwerkroutes (ontwerpredenering, voor het uitgestelde DC-LAN-werk):** Netwerkroutes omzeilen ACL policy standaard. Netwerken vereisen groepkoppeling by design. DC-LAN zou Netwerken moeten gebruiken voor zero-trust-correctheid; het exit node moet Netwerkroutes gebruiken (NetBird-beperking voor 0.0.0.0/0).

**Distribution Groups vs. ACL policy:** Deze dienen verschillende doelen: de distribution group bepaalt *welke peers de DNS-/netwerkconfiguratie ontvangen*, terwijl de ACL policy bepaalt *of het verkeer daadwerkelijk stroomt*. Als een peer in de distribution group zit maar geen ACL policy heeft die verkeer toestaat, ontvangt hij de config maar kan hij de resource niet bereiken. Symptoom: `netbird status` toont `Nameservers: 0/1 Available` op een peer die de nameserverconfig ontvangt maar er geen ACL-pad naartoe heeft.

### DNS-configuratie

NetBird Dashboard → DNS:

```
Aangepaste DNS-zone:   sandbox.local → pop01 (100.70.154.79)
Primaire nameserver: pop01 (100.70.154.79), Overeenkomende domeinen: (leeg)
```

De lege instelling voor overeenkomende domeinen routeert **alle** DNS-query's van NetBird-clients via pop01 Unbound, niet alleen `*.sandbox.local`. Zonder dit gebruiken externe query's de standaard-DNS van de client-adapter en omzeilen ze Unbound RPZ volledig. Zie [Bevinding: NetBird primaire nameserver](../findings/netbird-primary-nameserver.md).

---

## Integratiepunten

| Component | Richting | Wat |
|-----------|----------|-----|
| [Squid](squid.md) | transport | Overlay-IP `100.70.154.79` van wt0 moet bestaan voor Squid pre-auth listener |
| [ioc2rpz/RPZ](ioc2rpz.md) | DNS-afhankelijkheid | Primaire nameserver-instelling routeert alle client-DNS via Unbound RPZ |
| [Caddy](caddy.md) | → browser | WPAD PAC-bestand geleverd door Caddy op mgmt01 overlay-IP, alleen bereikbaar via NetBird |
| [Suricata](suricata.md) | zichtbaarheid | Suricata ziet WireGuard als versleuteld UDP op vtnet0; binnenste payload niet inspecteerbaar |
| [Drie-gate model](../decisions/ca-posture-hybrid.md) | authenticatielaag | Gate 1 (Entra ID CA) + Gate 2 (posture checks) koppelen beide aan de NetBird-aanmeldflow |

ClamAV/Python DLP ICAP-verkeer van pop01 naar mgmt01 loopt via het `192.168.122.0/24` WAN-segment, niet via de NetBird-overlay. Dit communicatiepad werkt ook wanneer de NetBird-tunnel niet actief is.

---

## JWT-groepsynchronisatie

De identiteitsketen loopt van Entra ID via Zitadel naar NetBird:

1. **Entra ID** persona-groepen (`2ITCSC1A-Studenten`, `2ITCSC1A-Docenten`, `2ITCSC1A-Admins`) → `cloud_displayname` optionalClaim in JWT
2. **Zitadel Action 1** (External Auth): allowlist-map weergavenaam → clean naam (`Studenten`, `Docenten`, `Admins`) → schrijft naar user metadata `sase_groups`
3. **Zitadel Action 2** (Complement Token): leest metadata → `setClaim("groups", [...])` in JWT
4. **NetBird JWT sync**: `groups`-claim wordt user auto-groups → gepropageerd naar alle peers van die gebruiker

**Persona-groepen:** `Studenten`, `Docenten`, `Admins` (clean namen; Entra-namen dragen `2ITCSC1A-`-prefix)

**JWT allow-groups veld:** Moet **leeg** gelaten worden. Een gevuld veld met een niet-matchende waarde veroorzaakt 401-lockout voor alle gebruikers; herstel vereist directe SQLite-toegang op de management-container. Zie [Bevinding: NetBird JWT allow-groups lockout](../findings/netbird-jwt-allow-groups-lockout.md).

**Service-user:** `identity-bridge` met admin-role voor API-toegang. Gewone user-PAT's triggeren issue #3127 (stript JWT-gepropageerde auto-groups van alle peers). Zie [Bevinding: NetBird issue #3127](../findings/netbird-issue-3127.md).

---

## NATS-integratie

NetBird is zowel producer als doelwit voor NATS-gestuurde handhaving:

- **Producer:** De [Identity Bridge](identity-bridge.md) (die afhankelijk is van de NetBird Management API) publiceert identity events naar `identity.peer.connected`, `identity.peer.disconnected` en `identity.multi_persona` (zero-trust-anomalie) wanneer peers verbinden, verbreken of in meer dan één personagroep verschijnen.
- **Handhavingsdoelwit:** De [Control Daemon](control-daemon.md) gebruikt de NetBird Groups API om peers in quarantaine te plaatsen door ze te verwijderen uit policy-bearing personagroepen (Studenten/Docenten/Admins). Onder het deny-by-default-model (default all-to-all policy verwijderd) verliest een peer zonder groepslidmaatschap alle connectiviteit.

---

## Bekende problemen / valkuilen

**`config.json` wordt 0 bytes na onzuivere afsluiting:** FreeBSD UFS met soft updates kan een leeg bestand achterlaten wanneer de schrijfbuffer niet wordt geleegd vóór QEMU-kill. NetBird start niet; wt0-interface bestaat niet; alle overlay-afhankelijke services breken. Beperking: `cp /var/db/netbird/config.json /var/db/netbird/config.json.bak` na elke sessie. Zie [Bevinding: NetBird config nul bytes](../findings/netbird-config-zero-bytes.md).

**OPNsense WebUI-vergrendeling na wijzigingen toepassen:** sommige firewallwijzigingen produceren een vergrendeling waarbij pf-regels tijdelijk onjuist zijn. Oplossing: `configctl filter reload` vanuit de OPNsense-shell, niet `pfctl -f`. Dit herstelt de door OPNsense gegenereerde regels correct.

**Docker-volumes vereisen containerrecreatie:** `docker compose restart caddy` past geen nieuwe volume-mounts toe. Gebruik in plaats daarvan `docker compose up -d caddy`.

**Zitadel-groepenclaim-mismatch:** Zitadel gebruikt geneste `roles` JSON; NetBird verwacht een platte `groups`-array. Als groepen niet propageren naar NetBird, controleer dan het Zitadel-actiescript voor groepenclaim-transformatie. Zie NetBird Zitadel-documentatie.

**`process_check` platformbeperkingen:** de `process_check`-posture check van NetBird is niet beschikbaar op iOS/Android. Bij het configureren van `os_version_check` geldt: als er voor een bepaald OS geen pad is opgegeven, wordt dat OS **standaard geblokkeerd** (niet toegestaan). Dit is een beveiligingspositieve standaard, maar kan onbedoeld hele platformen buitensluiten. Merk op dat procescontroles spoofbaar zijn: een dummy-binary op het verwachte pad voldoet aan de controle zonder dat de beveiligingssoftware daadwerkelijk draait.

---

## Gerelateerd

- [Architectuuroverzicht](../overview/architecture.md)
- [Concept: Zero Trust](../concepts/zero-trust.md)
- [Concept: SASE](../concepts/sase.md)
- [Beslissing: Zitadel als IdP-broker](../decisions/zitadel-idp-broker.md)
- [Beslissing: CA + posture hybride](../decisions/ca-posture-hybrid.md)
- [Beslissing: GNS3 vs. EVE-NG](../decisions/gns3-vs-eveng.md)
- [Bevinding: NetBird primaire nameserver](../findings/netbird-primary-nameserver.md)
- [Bevinding: NetBird config nul bytes](../findings/netbird-config-zero-bytes.md)
- [Runbook: ZTNA-overlay](../runbooks/02-ztna-overlay.md)
- [Identity Bridge](identity-bridge.md)
- [NATS JetStream](nats-jetstream.md)
- [Control Daemon](control-daemon.md)
- [Wazuh](wazuh.md)
- [Concept: Identity Flow](../concepts/identity-flow.md)
- [Runbook: Access Policy](../runbooks/07-access-policy.md)

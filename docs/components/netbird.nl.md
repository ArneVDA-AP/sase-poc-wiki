---
title: "NetBird + Zitadel + Entra ID — ZTNA-overlay"
tags: [netbird, zero-trust, sase, network, opnsense, docker]
---

# NetBird + Zitadel + Entra ID — ZTNA-overlay

**Rol:** ZTNA-transportlaag — creëert een WireGuard-mesh-overlay die BYOD-clients toegang geeft tot pop01 (proxy, DNS) en DC-LAN-resources, ongeacht hun fysieke netwerklocatie. Elke toegangsbeslissing is identiteitsgebaseerd; laterale beweging tussen resources vereist expliciet ACL-beleid.  
**Status:** ✅ Volledig operationeel — snapshot `Fase2-ZTNA-Complete`  
**Configuratielocatie:** mgmt01 Docker Compose (`~/docker-compose.yml`), NetBird Dashboard (`https://netbird.sandbox.local`)

---

## Werking in deze stack

Zonder NetBird heeft mobile01 geen netwerkpad naar pop01. De gehele SWG-pijplijn (Squid, ClamAV, DLP, Unbound RPZ) is afhankelijk van mobile01 dat de overlay-IP `100.70.154.79:3128` van pop01 bereikt. NetBird creëert de versleutelde tunnel die dit mogelijk maakt vanuit elk netwerk — thuis, mobiel, school — zonder enige service publiek bloot te stellen.

**De IdP-keten:**

```
Gebruiker → NetBird Dashboard → Zitadel (OIDC-uitgever, zelf-gehost op mgmt01)
                                        ↓
                               Entra ID (externe IdP via OIDC-federatie)
                                        ↓
                                Microsoft-inlogpagina (aplab.be-tenant)
```

Het quickstart-script van NetBird installeert Zitadel als primaire OIDC-uitgever. Entra ID is verbonden als externe IdP *aan Zitadel*, niet rechtstreeks aan NetBird. Dit is een architecturele realiteit van het quickstart-script, geen beperking — Zitadel voegt een centrale gebruikersbeheerlaag toe met rollen en groepen die onafhankelijk zijn van Entra ID-configuratie.

**Entra ID Conditional Access** evalueert bij het Entra ID `/authorize`-eindpunt, gericht op de NetBird-app-registratie `2ITCSC1A-Netbird-Sandbox` (`11803ee8-eb15-462c-a286-5415c17a29c6`). Of de OIDC-omleiding afkomstig is van Zitadel of rechtstreeks van een NetBird-client is irrelevant — de gebruiker authenticeert zich rechtstreeks bij Entra ID en CA wordt geactiveerd voor die app-registratie. Zie [Beslissing: CA-postuur hybride](../decisions/ca-posture-hybrid.md).

**WireGuard-mesh vs. per-app-tunnels:** NetBird creëert een volledige WireGuard-mesh waarbij ACL-beleid bepaalt welke peers kunnen communiceren. Dit is architectureel equivalent aan per-app Zero Trust-tunnels (Zscaler ZPA): een peer met toegang tot dc01 kan mgmt01 niet bereiken zonder een apart ACL-beleid — resources zijn niet zichtbaar, niet alleen ontoegankelijk.

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

Het script mislukt omdat Let's Encrypt geen certificaten kan uitreiken voor privé-hostnamen. Nadat het script de Docker Compose-stack aanmaakt, patch handmatig het Caddyfile:

```caddyfile
netbird.sandbox.local {
    tls internal    # ← voeg deze regel toe
    # ... bestaande reverse_proxy-configuratie
}
```

Voeg een Docker-netwerkalias toe zodat interne containers de hostnaam kunnen oplossen:

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

Elke machine die communiceert met NetBird moet de hostnaam kunnen oplossen:

| Machine | Methode |
|---------|---------|
| mgmt01 | `/etc/hosts`: `192.168.122.23 netbird.sandbox.local` |
| pop01 (OPNsense) | Console optie 8: `echo "192.168.122.23 netbird.sandbox.local" >> /etc/hosts` |
| mobile01 (Windows) | `C:\Windows\System32\drivers\etc\hosts` (als beheerder) |

De GNS3-host draait nginx SNI stream passthrough: `netbird.sandbox.local:443` → `192.168.122.23:443`.

### Entra ID externe IdP

Via Zitadel-console op `https://netbird.sandbox.local/zitadel/ui/console`:

```
Instellingen → Identity Providers → Toevoegen → Microsoft Azure AD
Client ID:     11803ee8-eb15-462c-a286-5415c17a29c6
Tenant ID:     23e9bcdc-5cb9-4867-9310-76cc0b462ddc
Client Secret: <uit Entra ID-app-registratie>
```

Voeg de door Zitadel gegenereerde omleidings-URI toe aan de Entra ID-app-registratie → Authenticatie.

**Gebruikersgoedkeuring:** Nieuwe gebruikers die via Entra ID authenticeren, verschijnen als "in behandeling" in NetBird Dashboard → Gebruikers. Een beheerder moet ze handmatig goedkeuren — dit is een bewuste beveiligingsfunctie van de Zitadel-quickstart, geen bug.

### Groepen

| Groep | Leden | Doel |
|-------|-------|------|
| `SASE-Admins` | pop01, mgmt01 | Administratieve SSH/HTTPS-toegang |
| `SASE-MobileUsers` | mobile01 | BYOD-clients |
| `SASE-Services` | pop01, mgmt01 | Service-eindpunten (proxy, WPAD, DNS) |
| `SASE-InternalResources` | (resourcegroep voor netwerk) | DC-LAN-resources |

**Volgorde van beleidsaanmaak is cruciaal:** Maak alle beleidsregels aan vóór het verwijderen van het standaard alles-naar-alles-beleid. Het verwijderen van de standaard zonder vervangers verbreekt onmiddellijk alle peer-communicatie.

### ACL-beleid

**Beleid 1 — Admin-Infrastructuur:** Bronnen: `SASE-Admins` → Bestemmingen: `SASE-Admins`, Protocol: Alles

**Beleid 2 — Mobiel-naar-Services:** Bronnen: `SASE-MobileUsers` → Bestemmingen: `SASE-Services`, Protocol: Alles  
Dit staat mobile01 toe pop01-proxy (`100.70.154.79:3128`) en mgmt01 WPAD (`100.70.135.241:80`) te bereiken.

**Beleid 3 — Datacentertoegang:** Bronnen: `SASE-MobileUsers` → Bestemmingen: `SASE-InternalResources`, Protocol: Alles

### Exit-knooppunt en DC-LAN-routering

**Exit-knooppunt (internetverkeer)** — via Netwerkroutes:
```
Netwerk: 0.0.0.0/0, Routeerpeer: pop01, Groepen: SASE-MobileUsers
```

**DC-LAN (10.0.0.0/8)** — via Netwerken (ACL-bewust, niet Netwerkroutes):
```
Netwerknaam: Internal-DC
Resource: DC-LAN, Type: IP-bereik, Bereik: 10.0.0.0/8, Groep: SASE-InternalResources
```

Netwerkroutes omzeilen ACL-beleid standaard. Netwerken vereisen groepkoppeling by design. DC-LAN moet Netwerken gebruiken voor zero-trust-correctheid; exit-knooppunt moet Netwerkroutes gebruiken (NetBird-documentatiebeperking voor 0.0.0.0/0).

### DNS-configuratie

NetBird Dashboard → DNS:

```
Aangepaste DNS-zone:   sandbox.local → pop01 (100.70.154.79)
Primaire naamserver: pop01 (100.70.154.79), Overeenkomende domeinen: (leeg)
```

De lege instelling voor overeenkomende domeinen routeert **alle** DNS-query's van NetBird-clients via pop01 Unbound — niet alleen `*.sandbox.local`. Zonder dit gebruiken externe query's de standaard-DNS van de client-adapter en omzeilen ze Unbound RPZ volledig. Zie [Bevinding: NetBird primaire naamserver](../findings/netbird-primary-nameserver.md).

---

## Integratiepunten

| Component | Richting | Wat |
|-----------|----------|-----|
| [Squid](squid.md) | transport | Overlay-IP `100.70.154.79` van wt0 moet bestaan voor Squid pre-auth listener |
| [ioc2rpz/RPZ](ioc2rpz.md) | DNS-afhankelijkheid | Primaire naamserverinstelling routeert alle client-DNS via Unbound RPZ |
| [Caddy](caddy.md) | → browser | WPAD PAC-bestand geleverd door Caddy op mgmt01 overlay-IP, alleen bereikbaar via NetBird |
| [Suricata](suricata.md) | zichtbaarheid | Suricata ziet WireGuard als versleuteld UDP op vtnet0 — binnenste payload niet inspecteerbaar |
| [Driepoortmodel](../decisions/ca-posture-hybrid.md) | authenticatielaag | Poort 1 (Entra ID CA) + Poort 2 (postuurcontroles) koppelen beide aan de NetBird-aanmeldflow |

ClamAV/Python DLP ICAP-verkeer van pop01 naar mgmt01 loopt via het `192.168.122.0/24` WAN-segment, niet via de NetBird-overlay. Dit communicatiepad werkt ook wanneer de NetBird-tunnel niet actief is.

---

## JWT-groepsynchronisatie

De identiteitsketen loopt van Entra ID via Zitadel naar NetBird:

1. **Entra ID** persona-groepen (`2ITcsc1A-Studenten`, `2ITcsc1A-Docenten`, `2ITcsc1A-Admins`) → `cloud_displayname` optionalClaim in JWT
2. **Zitadel Action 1** (External Auth): allowlist-map GUID → clean naam (`Studenten`, `Docenten`, `Admins`) → schrijft naar user metadata `sase_groups`
3. **Zitadel Action 2** (Complement Token): leest metadata → `setClaim("groups", [...])` in JWT
4. **NetBird JWT sync**: `groups`-claim wordt user auto-groups → gepropageerd naar alle peers van die gebruiker

**Persona-groepen:** `Studenten`, `Docenten`, `Admins` (clean namen; Entra-namen dragen `2ITcsc1A-`-prefix)

**JWT allow-groups veld:** Moet **leeg** gelaten worden. Een gevuld veld met een niet-matchende waarde veroorzaakt 401-lockout voor alle gebruikers — herstel vereist directe SQLite-toegang op de management-container. Zie [Bevinding: NetBird JWT allow-groups lockout](../findings/netbird-jwt-allow-groups-lockout.md).

**Service-user:** `identity-bridge` met admin-role voor API-toegang. Gewone user-PAT's triggeren issue #3127 (stript JWT-gepropageerde auto-groups van alle peers). Zie [Bevinding: NetBird issue #3127](../findings/netbird-issue-3127.md).

**Groepsmigratie (V34):** Alle SASE-MobileUsers/SiteUsers/etc. vervangen door persona-groepen + Core-Services.

---

## NATS-integratie

NetBird is zowel producer als doelwit voor NATS-gestuurde handhaving:

- **Producer:** De [Identity Bridge](identity-bridge.md) (die afhankelijk is van de NetBird Management API) publiceert identiteitsgebeurtenissen naar `identity.login` en `identity.group_change` wanneer peerverbindingen of groepslidmaatschappen wijzigen.
- **Handhavingsdoelwit:** De [Control Daemon](control-daemon.md) gebruikt de NetBird Groups API om peers in quarantaine te plaatsen door ze te verwijderen uit beleidsdragende personagroepen (Studenten/Docenten/Admins). Onder het deny-by-default-model (standaard All->All-beleid verwijderd) verliest een peer zonder groepslidmaatschap alle connectiviteit.

---

## Bekende problemen / valkuilen

**`config.json` wordt 0 bytes na onzuivere afsluiting** — FreeBSD UFS met soft updates kan een leeg bestand achterlaten wanneer de schrijfbuffer niet wordt geleegd vóór QEMU-kill. NetBird start niet; wt0-interface bestaat niet; alle overlay-afhankelijke services breken. Beperking: `cp /var/db/netbird/config.json /var/db/netbird/config.json.bak` na elke sessie. Zie [Bevinding: NetBird config nul bytes](../findings/netbird-config-zero-bytes.md).

**OPNsense WebUI-vergrendeling na wijzigingen toepassen** — sommige firewallwijzigingen produceren een vergrendeling waarbij pf-regels tijdelijk onjuist zijn. Oplossing: `configctl filter reload` vanuit de OPNsense-shell, niet `pfctl -f`. Dit herstelt de door OPNsense gegenereerde regels correct.

**Docker-volumes vereisen containerrecreatie** — `docker compose restart caddy` past geen nieuwe volume-mounts toe. Gebruik in plaats daarvan `docker compose up -d caddy`.

**Zitadel-groepenclaim-mismatch** — Zitadel gebruikt geneste `roles` JSON; NetBird verwacht een platte `groups`-array. Als groepen niet propageren naar NetBird, controleer dan het Zitadel-actiescript voor groepenclaim-transformatie. Zie NetBird Zitadel-documentatie.

---

## Gerelateerd

- [Architectuuroverzicht](../overview/architecture.md)
- [Concept: Zero Trust](../concepts/zero-trust.md)
- [Concept: SASE](../concepts/sase.md)
- [Beslissing: Zitadel als IdP-broker](../decisions/zitadel-idp-broker.md)
- [Beslissing: CA + postuur hybride](../decisions/ca-posture-hybrid.md)
- [Beslissing: GNS3 vs. EVE-NG](../decisions/gns3-vs-eveng.md)
- [Bevinding: NetBird primaire naamserver](../findings/netbird-primary-nameserver.md)
- [Bevinding: NetBird config nul bytes](../findings/netbird-config-zero-bytes.md)
- [Identity Bridge](identity-bridge.md)
- [NATS JetStream](nats-jetstream.md)
- [Control Daemon](control-daemon.md)
- [Wazuh](wazuh.md)
- [Concept: Identity Flow](../concepts/identity-flow.md)

---
title: "Runbook: ZTNA Overlay"
tags: [runbook, netbird, zitadel, entra-id, wireguard, ztna]
---

# Runbook: ZTNA Overlay

**Bron:** `raw/Doc6_NetBird_ZTNA.md`
**Node(s):** mgmt01 (Docker-stack) + pop01 (NetBird-agent) + alle peers
**Vereisten:** [Runbook 01: Labomgeving](01-lab-environment.nl.md) afgerond, Entra ID-tenant met A5-licentie
**Status:** Operationeel — snapshot `Fase2-ZTNA-Complete`

---

## Vereistenchecklist

- [ ] Labomgeving volledig operationeel (Runbook 01)
- [ ] Entra ID-tenant (`aplab.be`) toegankelijk
- [ ] A5 Educational-licentie actief
- [ ] App-registratie aangemaakt in Entra ID:
  - Client-ID: `cebe0d74-be9f-49ac-9f35-65f11586c1bb`
  - Tenant-ID: `23e9bcdc-5cb9-4867-9310-76cc0b462ddc`
- [ ] mgmt01 heeft Docker geïnstalleerd en draaiend

---

## Stap 1: Entra ID App-registratie

Maak de app-registratie aan in Entra ID vóór het uitrollen van NetBird:

```
Entra ID → App Registrations → New Registration
  Naam: NetBird SASE PoC
  Ondersteunde accounttypen: Accounts in this organizational directory only
```

Noteer de Client-ID en Tenant-ID. De redirect-URI wordt later toegevoegd (na NetBird-deployment, Stap 8).

---

## Stap 2: Hosts-vermelding op mgmt01

```bash
ssh mgmt@10.158.10.67 -p 7023
```

```bash
echo "192.168.122.23  netbird.sandbox.local" | sudo tee -a /etc/hosts
ping -c 2 netbird.sandbox.local
# Moet resolven naar 192.168.122.23
```

---

## Stap 3: Quickstart-script uitvoeren

```bash
curl -sSLO https://github.com/netbirdio/netbird/releases/latest/download/getting-started-with-zitadel.sh
chmod +x getting-started-with-zitadel.sh
sudo ./getting-started-with-zitadel.sh
```

Wanneer gevraagd:
```
Enter the domain you want to use for NetBird: netbird.sandbox.local
Select your Identity Provider setup: [0] None (use Zitadel built-in)
```

> **Valkuil: Het script zal falen — dit is verwacht.** Het probeert Let's Encrypt te contacteren voor `netbird.sandbox.local` en de ACME-uitdaging mislukt (privéhostnaam). Stop met **Ctrl+C** en ga door naar Stap 4.

---

## Stap 4: Caddyfile aanpassen voor `tls internal`

```bash
nano Caddyfile
```

Voeg `tls internal` toe aan elk serverblok dat verwijst naar `netbird.sandbox.local`:

```caddyfile
netbird.sandbox.local {
    tls internal

    # ... bestaande reverse_proxy-configuratie
}

netbird.sandbox.local:443 {
    tls internal

    # ... bestaande configuratie
}
```

Zie [Beslissing: Zitadel als IdP-broker](../decisions/zitadel-idp-broker.nl.md) waarom zelfondertekend TLS hier acceptabel is.

---

## Stap 5: Docker-netwerkalias toevoegen

```bash
nano docker-compose.yml
```

Zoek de `caddy`-service en voeg een netwerkalias toe:

```yaml
services:
  caddy:
    networks:
      netbird:
        aliases:
          - netbird.sandbox.local
```

Hierdoor kunnen interne containers (Zitadel, management) de hostnaam via deze alias resolven.

---

## Stap 6: Stack herstarten en valideren

```bash
sudo docker compose down
sudo docker compose up -d
```

Wacht 60 seconden tot alle services zijn geïnitialiseerd:

```bash
sudo docker compose ps
# Alle containers moeten status "Up" tonen

curl -k https://netbird.sandbox.local/zitadel/debug/ready
# Moet "ok" of HTTP 200 teruggeven
```

Controleer `management.json` — alle URL-verwijzingen moeten `https://netbird.sandbox.local` gebruiken, niet bare IP's. Controleer deze velden:

```bash
cat management.json | grep -i "netbird.sandbox.local"
# Te controleren velden:
#   HttpConfig.AuthIssuer          → moet netbird.sandbox.local bevatten
#   IdpManagerConfig.ClientConfig.Issuer → moet netbird.sandbox.local bevatten
```

> **Valkuil: Docker volume-mounts vereisen containerrecreatie, niet herstart.** `docker compose restart caddy` past nieuwe volume-mounts NIET toe. Gebruik altijd `docker compose up -d caddy`.
> Zie [Finding: Docker volume-recreatie](../findings/docker-volume-recreation.nl.md).

---

## Stap 7: Hosts-vermeldingen op alle NetBird-clients

Elke machine die communiceert met NetBird moet de hostnaam kunnen resolven:

| Machine | Methode |
|---------|---------|
| pop01 (OPNsense) | Console → optie 8 (shell): `echo "192.168.122.23 netbird.sandbox.local" >> /etc/hosts` |
| mobile01 (Windows) | Als Administrator: bewerk `C:\Windows\System32\drivers\etc\hosts`, voeg `192.168.122.23 netbird.sandbox.local` toe |
| site01 (VyOS) | `set system static-host-mapping host-name netbird.sandbox.local inet 192.168.122.23` |

Op de GNS3-host: `netbird.sandbox.local` wordt gerouteerd via de nginx SNI-stream passthrough → `192.168.122.23:443`.

**Verificatie:** Vanaf elke machine `ping netbird.sandbox.local` of equivalent resolveert correct.

---

## Stap 8: Entra ID koppelen als externe IdP in Zitadel

Navigeer naar de Zitadel-console:

```
https://netbird.sandbox.local/zitadel/ui/console
→ Settings → Identity Providers → Add → Microsoft Azure AD
→ Client ID: cebe0d74-be9f-49ac-9f35-65f11586c1bb
→ Client Secret: <vanuit Entra ID app-registratie>
→ Tenant ID: 23e9bcdc-5cb9-4867-9310-76cc0b462ddc
```

In Entra ID → App-registratie → Authentication: voeg de redirect-URI toe die Zitadel genereert.

**Gebruikersgoedkeuringsflow:** Nieuwe gebruikers die inloggen via Entra ID verschijnen als "pending" in NetBird Dashboard → Users. Een beheerder moet ze handmatig goedkeuren. Dit is een bewuste beveiligingsfunctie, geen fout.

**Groepssynchronisatie:** Zitadel gebruikt `roles` (geneste JSON) terwijl NetBird een platte `groups`-array verwacht. Als groepslidmaatschappen niet verschijnen, controleer het Zitadel Action-script voor groepsclaim-transformatie.

---

## Stap 9: NetBird-agent installeren op pop01

Installeer vanuit de pop01-console (FreeBSD) de NetBird-client. De WireGuard-interface `wt0` wordt automatisch aangemaakt.

> **Valkuil: `config.json` wordt 0 bytes na onregelmatige afsluiting.** FreeBSD's UFS met soft updates kan een leeg bestand achterlaten wanneer QEMU abrupt wordt beëindigd. NetBird start niet, `wt0` bestaat niet, en alle overlay-listeners falen.
> Zie [Finding: NetBird config nul bytes](../findings/netbird-config-zero-bytes.nl.md).

**Maatregel — maak na elke sessie een back-up:**

```bash
cp /var/db/netbird/config.json /var/db/netbird/config.json.bak
```

Herstel: als `config.json` 0 bytes is, herstel vanuit de back-up en herstart NetBird.

---

## Stap 10: Groepen aanmaken in NetBird Dashboard

NetBird Dashboard → Peers → maak groepen aan:

| Groep | Leden | Rol |
|-------|-------|-----|
| `SASE-Admins` | pop01, mgmt01 | Administratieve SSH/HTTPS-toegang |
| `SASE-MobileUsers` | mobile01, toekomstige BYOD | BYOD-clients |
| `SASE-Services` | pop01, mgmt01 | Service-eindpunten (WPAD, DNS, proxy) |
| `SASE-InternalResources` | (resourcegroep voor netwerk) | DC-LAN-resources |

mgmt01 zit in zowel `SASE-Admins` als `SASE-Services` — dit scheidt de infrarol (beheertoegang) van de servicerol (WPAD via Caddy, ioc2rpz-feeds).

---

## Stap 11: ACL-beleidsregels aanmaken

> **Valkuil: Volgorde is cruciaal.** Maak alle beleidsregels aan **vóór** het verwijderen van het standaard alles-naar-alles-beleid. Als je het standaardbeleid verwijdert zonder vervangers, valt alle peercommunicatie direct weg.

**Beleid 1 — Admin-Infrastructuur:**

```
Naam:        Admin-Infrastructure
Bronnen:     SASE-Admins
Bestemmingen: SASE-Admins
Protocol:    All
Actie:       Accept
```

**Beleid 2 — Mobiel-naar-Services:**

```
Naam:        Mobile-to-Services
Bronnen:     SASE-MobileUsers
Bestemmingen: SASE-Services
Protocol:    All
Actie:       Accept
```

**Beleid 3 — Datacentertoegeng:**

```
Naam:        Datacenter Access
Bronnen:     SASE-MobileUsers
Bestemmingen: SASE-InternalResources
Protocol:    All
Actie:       Accept
```

---

## Stap 12: Exit node configureren via Network Routes

NetBird Dashboard → Network Routes → Add Route:

```
Netwerk:      0.0.0.0/0
Routing Peer: pop01
Beschrijving: Internet exit node
Groepen:      SASE-MobileUsers
```

Dit routeert al het niet-overlay-verkeer van BYOD-clients via pop01 — vereist zodat Squid het verkeer kan inspecteren.

---

## Stap 13: DC-LAN configureren via Networks

NetBird Dashboard → Networks → Create Network:

```
Naam: Internal-DC
```

Voeg routing peer toe: pop01. Voeg resource toe:

```
Naam:   DC-LAN
Type:   IP Range
Bereik: 10.0.0.0/8
Groep:  SASE-InternalResources
```

Zie [Beslissing: CA + Posture hybride](../decisions/ca-posture-hybrid.nl.md) waarom Networks (ACL-bewust) wordt gebruikt voor DC-LAN in plaats van Network Routes.

**Verificatie:** Network Routes-tabel toont `Internal-DC` als actief met pop01 online (groene indicator).

---

## Stap 14: NetBird DNS configureren

NetBird Dashboard → DNS:

**Aangepaste DNS-zone:**
```
Domein:       sandbox.local
Nameserver:   pop01 (100.70.154.79)
```

**Primaire nameserver:**
```
Primaire nameserver: pop01 (100.70.154.79)
Match-domeinen:      (LEEG LATEN)
```

> **Valkuil: Lege match-domeinen is cruciaal.** Leeg betekent dat pop01 de primaire nameserver wordt voor **alle** DNS-queries — niet alleen `*.sandbox.local`. Zonder dit omzeilen externe queries Unbound en biedt DNS RPZ geen bescherming voor externe domeinen.
> Zie [Finding: NetBird primaire nameserver](../findings/netbird-primary-nameserver.nl.md).

---

## Stap 15: Valideren op mobile01

```powershell
# NetBird-status
netbird status
# Verwacht: Connected, X/Y peers, routes active

# Datacenter bereikbaar
Test-NetConnection 10.0.0.100 -Port 80
# Verwacht: TcpTestSucceeded: True

# Internet via exit node
Test-NetConnection 8.8.8.8
# Verwacht: PingSucceeded: True
```

---

## Eindverificatie

- [ ] `curl -k https://netbird.sandbox.local/zitadel/debug/ready` geeft HTTP 200
- [ ] Browser naar `https://netbird.sandbox.local` toont Zitadel-inlogpagina
- [ ] Inloggen met Entra ID-testaccount werkt (MFA-prompt verschijnt)
- [ ] NetBird Dashboard toegankelijk, peers zichtbaar (groen = online)
- [ ] `netbird status` op mobile01 toont "Connected"
- [ ] Ping tussen overlay-IP's: `100.70.154.79`, `100.70.135.241`, `100.70.95.98`
- [ ] DC-LAN bereikbaar vanuit mobile01 (`10.0.0.100`)
- [ ] Internet bereikbaar vanuit mobile01 via exit node
- [ ] `config.json` back-up aangemaakt op pop01

---

## Gerelateerd

- [Component: NetBird](../components/netbird.nl.md)
- [Component: Caddy](../components/caddy.nl.md)
- [Concept: Zero Trust](../concepts/zero-trust.nl.md)
- [Beslissing: Zitadel als IdP-broker](../decisions/zitadel-idp-broker.nl.md)
- [Finding: NetBird config nul bytes](../findings/netbird-config-zero-bytes.nl.md)
- [Finding: Docker volume-recreatie](../findings/docker-volume-recreation.nl.md)
- [Finding: NetBird primaire nameserver](../findings/netbird-primary-nameserver.nl.md)

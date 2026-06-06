---
title: "Runbook: Cosmos Application Gateway"
tags: [runbook, cosmos, application-gateway, mfa, smartshield, docker, dns, netbird]
---

# Runbook: Cosmos Application Gateway

**Bron:** `rayan_Cosmos Implementatie SASE PoC.md`, `DEMO_SCRIPT_COSMOS_MFA.md`
**Node(s):** dc01 (`10.0.0.100`) + OPNsense (Unbound, `wt0`) + NetBird Dashboard + Windows-client
**Vereisten:** NetBird-overlay operationeel met `10.0.0.0/8` geadverteerd aan peers ([Runbook 02: ZTNA Overlay](02-ztna-overlay.nl.md)); dc01 bereikbaar over de overlay
**Status:** ✅ PoC-gevalideerd op de parallelle stack, sandbox-integratie pending

> **Scope.** Deze runbook richt zich op de parallelle `SASE_POC`-omgeving (dc01 `10.0.0.100`), niet op de main sandbox. Cosmos is een additieve application-admission-laag bovenop NetBird; waar het overlapt met de sandbox (DNS, de overlay) is de sandbox de bron van waarheid. Toegang tot dc01 in deze omgeving loopt via de jump host: `ssh -J root@10.158.10.67:9022 dc@10.0.0.100`.

---

## Vereistenchecklist

- [ ] dc01 draait Ubuntu 22.04+ (build gebruikt 24.04.4 LTS), 2+ vCPU, 3 GB+ RAM, statisch IP `10.0.0.100/24`, gateway `10.0.0.1` (OPNsense LAN)
- [ ] dc01 heeft internet via OPNsense + NAT (voor Docker image pulls)
- [ ] Poort 80 en 443 vrij op dc01, geen andere webserver mag draaien
- [ ] DNS-resolutie via Unbound op OPNsense (`10.0.0.1`)
- [ ] NetBird-overlay up; dc01 bereikbaar vanaf een NetBird-client (Runbook 02)
- [ ] Authenticator-app klaar (Microsoft / Google Authenticator)

---

## Stap 1: DC-1-voorbereiding

Verbind met dc01 via de jump host:

```bash
ssh -J root@10.158.10.67:9022 dc@10.0.0.100
```

> **Valkuil: nginx moet gemaskerd worden, niet enkel gestopt.** Ubuntu Server kan nginx als dependency installeren; het grijpt poort 80/443 die Cosmos nodig heeft, en `systemctl stop` is niet genoeg omdat nginx na een reboot herstart.

```bash
sudo systemctl stop nginx
sudo systemctl disable nginx
sudo systemctl mask nginx
sudo systemctl status nginx     # moet tonen: masked
```

Installeer Docker (officiële repo) en pin daarna de daemon-config; beide instellingen hieronder zijn verplicht voor deze GNS3-omgeving:

> **Valkuil: twee kritieke daemon.json-instellingen samen.** Een **custom address pool** houdt Docker-bridges weg van de GNS3-ranges (`10.0.0.0/8`, `192.168.122.0/24`), en **MTU 1400** laat ruimte voor GNS3/KVM-encapsulation-overhead. Standaard MTU 1500 veroorzaakt packet-fragmentatie en mysterieus verlies van connectiviteit.

```bash
sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "default-address-pools": [ { "base": "192.168.200.0/20", "size": 24 } ],
  "mtu": 1400,
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}
EOF
sudo systemctl restart docker
```

**Verificatie:**

```bash
sudo docker info | grep -A 3 "Default Address Pools"   # moet 192.168.200.0/20 tonen
sudo docker info | grep MTU                            # moet 1400 tonen
```

---

## Stap 2: Cosmos installeren

> **Valkuil: geen Portainer / CasaOS / Unraid-templates.** De officiële Cosmos-docs zijn expliciet: die breken de configuratie. Gebruik altijd het `docker run`-commando hieronder.

```bash
sudo docker run -d \
  --network host \
  --privileged \
  --name cosmos-server \
  -h cosmos-server \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/run/dbus/system_bus_socket:/var/run/dbus/system_bus_socket \
  -v /:/mnt/host \
  -v /var/lib/cosmos:/config \
  azukaar/cosmos-server:latest
```

`--network host` laat Cosmos 80/443 op de host binden; `--privileged` + de Docker-socket laten het de andere containers beheren; `/var/lib/cosmos:/config` is persistente config, certs en DB-credentials.

**Verificatie (na ~30 s):**

```bash
sudo docker ps | grep cosmos-server
sudo docker logs cosmos-server --tail 20
```

Bereik de UI via een SSH-tunnel (DNS is nog niet geconfigureerd). In een apart terminal:

```bash
ssh -J root@10.158.10.67:9022 -L 8080:10.0.0.100:443 dc@10.0.0.100
```

Browser: `https://localhost:8080/cosmos-ui/`. De self-signed certificaatwaarschuwing is verwacht; klik erdoor.

---

## Stap 3: Initiële configuratie-wizard

Bij eerste keer openen draait Cosmos een setup-wizard. Stel in:

| Veld | Waarde | Reden |
|------|--------|-------|
| Hostname | `10.0.0.100` | Gesloten lab; zie de OAuth-kanttekening hieronder |
| HTTPS Mode | `SELFSIGNED` | Geen Let's Encrypt in een gesloten lab |
| Allow insecure via local IP | On | Admin-toegang via IP zonder certificaatwaarschuwing |
| Force Multi-Factor Authentication | On | Verplicht voor alle gebruikers |
| Admin username / password | `admin` / `<admin-password>` | |

> **Valkuil: hostname op IP beperkt OAuth/SSO (architecturaal).** Met de hostname als IP bouwt Cosmos zijn OAuth-redirect als `https://10.0.0.100/cosmos-ui/openid`; omdat cookies domain-gebonden zijn, is de `AuthEnabled`-toggle grijs voor `*.dc.local`-routes en werkt cross-domain SSO niet. Dit wordt in de PoC bewust aanvaard. Om later te migreren: voeg `cosmos.dc.local` toe aan dc01 `/etc/hosts`, wijzig de hostname in Configuration → General, herstart Cosmos. Zie [Bevinding: hostname op IP beperkt OAuth](../findings/cosmos-hostname-oauth.nl.md).

Na opslaan en inloggen als admin verschijnt een **New MFA Setup**-scherm met een QR-code. Open Microsoft / Google Authenticator → account toevoegen → scan → voer de 6-cijferige token in → Login. Bewaar deze app-entry: elke Cosmos-login heeft die nodig.

> **Valkuil: Constellation VPN is een betaalde functie in v0.22.10.** Activeer het niet; NetBird ZTNA vervangt het (NetBird adverteert `10.0.0.0/8` aan alle peers). En SmartShield heeft **geen globale toggle** in v0.22.10; het wordt per route geconfigureerd, automatisch bij installatie via Cosmos Market.

---

## Stap 4: DNS-configuratie

DNS moet op twee plaatsen worden gezet: Unbound op OPNsense (zodat clients resolven) en `/etc/hosts` op dc01 (zodat Cosmos zelf resolvet).

**Unbound host overrides.** OPNsense UI → Services → Unbound DNS → Overrides → Host Overrides → + Add:

| Host | Domain | IP |
|------|--------|----|
| `cosmos` | `dc.local` | `10.0.0.100` |
| `kuma` | `dc.local` | `10.0.0.100` |
| `gitea` | `dc.local` | `10.0.0.100` |

Save → Apply Changes.

> **Valkuil: de interne resolver van Cosmos kent `dc.local` niet.** De meest verrassende valkuil. Cosmos draait als Docker-container en gebruikt Docker's interne resolver (`127.0.0.11`), die `dc.local` niet kent. Een route op `kuma.dc.local` faalt met `lookup kuma.dc.local: no such host` tot de naam aan dc01's eigen hosts-bestand is toegevoegd (de container erft het).

```bash
sudo tee -a /etc/hosts <<'EOF'
10.0.0.100 gitea.dc.local
10.0.0.100 kuma.dc.local
10.0.0.100 cosmos.dc.local
EOF
cat /etc/hosts | grep dc.local   # moet alle drie tonen
```

Voeg elke nieuwe `*.dc.local`-service aan dit bestand toe zodra je hem deployt, anders geeft de route een internal server error.

---

## Stap 5: NetBird DNS-integratie

NetBird-verbonden clients moeten `*.dc.local` kunnen resolven. Dit vereist een NetBird nameserver-rule en een OPNsense-firewall rule.

**NetBird nameserver-rule.** NetBird management-UI → DNS → Nameservers → Add: Nameserver IP `10.0.0.1`, Match domains `dc.local`, Groups `All`. Dit zegt NetBird om `dc.local`-queries naar Unbound op OPNsense te sturen.

> **Valkuil: OPNsense blokkeert standaard DNS van de NetBird-overlay.** NetBird-verkeer komt binnen op de `wt0`-interface; OPNsense blokkeert poort 53 van niet-LAN-interfaces standaard, dus een firewall rule is vereist.

OPNsense UI → Firewall → Rules → WireGuard (`wt0`) → + Add: Action Pass, Protocol TCP/UDP, Source any, Destination This Firewall, Destination Port 53. Save → Apply.

> **Valkuil: het school-search-domein kaapt de resolutie.** Het AP Hogeschool-netwerk pusht een `bletchley.cloud`-search-domein via DHCP, waardoor Unbound het appendt (`kuma.dc.local.bletchley.cloud`) en naar een extern IP resolvet. Fix: OPNsense → Services → Unbound DNS → General → controleer het "Domain"-veld → verwijder `bletchley.cloud` als het er staat → Apply.

**Verificatie (Windows-client, NetBird verbonden):**

```powershell
netbird status                 # Status: Connected
nslookup kuma.dc.local         # moet resolven naar 10.0.0.100
ping 10.0.0.100                # moet antwoorden
```

---

## Stap 6: Services, Uptime Kuma (met MFA-gate)

Uptime Kuma is het monitoring-dashboard; we zetten het achter de Cosmos MFA-gate.

Cosmos UI → App Market → zoek "Uptime Kuma" → Install. Stel **Hostname** `kuma.dc.local` in; laat de rest standaard.

> **Valkuil: installeer de socket-proxy niet.** Cosmos Market kan naast Uptime Kuma een "socket-proxy" aanbieden; die veroorzaakt een link error en is niet nodig. Deactiveer die optie als hij aangeboden wordt.

Cosmos genereert de route-beveiligingsconfig automatisch (`SmartShield` aan, `AuthEnabled` true, `BlockCommonBots`, `ThrottlePerMinute` 12000, `cosmos-force-network-secured` true). Als de route na installatie nog `10.0.0.100` toont: Cosmos UI → URLs → Uptime-Kuma → Edit → Use Host On, Hostname `kuma.dc.local` → Save.

**Verificatie (Windows-client, NetBird verbonden):** surf naar `https://kuma.dc.local`. Verwachte flow: Cosmos-login (`admin` / `<admin-password>`) → MFA-code uit Authenticator → Kuma's eigen login → dashboard. Test met een **incognito-venster** om de gate te bewijzen; een actieve 48u-sessie zou anders het MFA-scherm overslaan (dit is session management, geen bypass).

### Gitea (zonder MFA-gate)

Gitea wordt bewust **zonder** de Cosmos MFA-gate gedeployed; het heeft zijn eigen volwassen auth, dus een tweede MFA-laag zou redundant zijn (zie [Beslissing: Twee-laags ZTNA](../decisions/cosmos-two-layer-ztna.nl.md)). Cosmos UI → App Market → "Gitea" → Install, Hostname `gitea.dc.local`. In Gitea's first-run-wizard: zet **SSH Server Port** op `222` (Docker bindt Gitea-SSH op 222) en laat alle database-/SQLite-paden ongemoeid.

> **Valkuil: Gitea-poort 3000 is niet extern gebonden.** Gitea's `3000/tcp` is opzettelijk niet blootgesteld; het is enkel bereikbaar via Cosmos. Gebruik voor een Uptime Kuma-monitor `https://10.0.0.100` (Cosmos), nooit `:3000`.

---

## Stap 7: CA-certificaat (optioneel, voor een propere demo)

Cosmos levert een self-signed cert; browsers waarschuwen tot de CA vertrouwd is. Om de waarschuwing op een client weg te halen, serveer `cosmos-ca.crt` vanaf dc01 en importeer het:

```bash
# Op dc01: tijdelijke HTTP-server in de map met cosmos-ca.crt
cd /home/dc && python3 -m http.server 8888
```

```powershell
# Op de Windows-client (NetBird verbonden):
Invoke-WebRequest -Uri http://10.0.0.100:8888/cosmos-ca.crt -OutFile C:\cosmos-ca.crt
certutil -addstore -f "Root" C:\cosmos-ca.crt
```

Stop de dc01 HTTP-server (`Ctrl+C`), sluit de browser volledig en heropen, surf naar `https://kuma.dc.local`; het slotje moet groen zijn zonder waarschuwing.

---

## Stap 8: MFA-enrollment-verificatie (demo-gate)

Bewijs de gate end-to-end vanuit een schone staat:

1. Open een incognito-/privévenster (geen actieve Cosmos-sessie).
2. Surf naar `https://kuma.dc.local` → Cosmos-loginpagina verschijnt (niet Kuma).
3. Voer `admin` / `<admin-password>` in → Login → Cosmos MFA-scherm verschijnt (Kuma nog niet zichtbaar).
4. Voer de 6-cijferige code uit Authenticator in → Cosmos redirect naar het Uptime Kuma-dashboard.

Dit is het twee-gate-model in actie: NetBird liet het device toe, Cosmos laat de gebruiker toe met MFA.

---

## Eindverificatie

- [ ] `sudo docker ps` toont `cosmos-server` (host 80+443), `cosmos-mongo-*`, `Uptime-Kuma` (healthy), `Gitea`
- [ ] `sudo docker info` bevestigt address pool `192.168.200.0/20` en MTU 1400
- [ ] nginx toont `masked`
- [ ] `nslookup kuma.dc.local` vanaf een NetBird-client resolvet naar `10.0.0.100`
- [ ] Incognito → `https://kuma.dc.local` toont de Cosmos-login, dan MFA, dan Kuma
- [ ] `https://gitea.dc.local` bereikt Gitea (geen Cosmos MFA-gate, by design)
- [ ] (Optioneel) CA geïmporteerd op de client; geen "Not secure"-waarschuwing
- [ ] Uptime Kuma-monitors gebruiken IP-adressen (bv. `https://10.0.0.100`), niet `*.dc.local`

---

## Gerelateerd

- [Component: Cosmos](../components/cosmos.nl.md)
- [Concept: Application Gateway](../concepts/application-gateway.nl.md)
- [Concept: Zero Trust](../concepts/zero-trust.nl.md)
- [Beslissing: Twee-laags ZTNA (NetBird + Cosmos)](../decisions/cosmos-two-layer-ztna.nl.md)
- [Bevinding: Cosmos hostname op IP beperkt OAuth/SSO](../findings/cosmos-hostname-oauth.nl.md)
- [Component: NetBird](../components/netbird.nl.md)
- [Runbook 02: ZTNA Overlay](02-ztna-overlay.nl.md)

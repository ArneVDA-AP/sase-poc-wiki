---
title: "Identity Bridge"
tags: [identity-bridge, netbird, squid, entra-id, fastapi, zero-trust]
---

# Identity Bridge

**Rol:** FastAPI-service op mgmt01 die NetBird WireGuard overlay-IP's in real-time koppelt aan Entra ID persona-groepen, zodat Squid identiteitsgebaseerde filtering kan uitvoeren.  
**Versie:** Python 3.12 (FastAPI 0.115.0, uvicorn 0.30.0, httpx 0.27.0)  
**Configuratielocatie:** mgmt01 Docker, `/opt/identity-bridge/`, gedeeld geheim in `.env`

## Hoe het werkt in deze stack

De Identity Bridge overbrugt de kloof tussen de ZTNA-transportlaag (NetBird) en de SWG-inspectielaag (Squid). Zonder deze bridge ziet Squid enkel overlay-IP-adressen en kan het niet bepalen welke gebruiker of persona-groep een verzoek indient.

De bridge pollt de NetBird Management API (`/api/peers` + `/api/users`) elke 30 seconden via een service-user PAT met admin-rol. Het bouwt een in-memory cache op die overlay-IP's koppelt aan persona-groepen. Squid bevraagt deze cache via `external_acl_type` bij elk HTTP-verzoek, met het overlay-IP van de client (`%SRC`) als opzoeksleutel.

De antwoordstroom: Squid stuurt `<IP> <group>` → helper bevraagt `GET /lookup?ip=<overlay_ip>` (met de `X-Bridge-Secret`-header) → Identity Bridge retourneert het volledige groepslidmaatschap van de peer → de helper antwoordt `OK user=<email>` als de gevraagde groep matcht, anders `ERR` → Squid's persona-ACL wordt geactiveerd of valt door. De bridge selecteert zelf geen persona; persona-resolutie gebeurt in de `http_access`-volgorde van Squid (V31).

## Configuratie

- **Service-gebruiker:** `identity-bridge` met admin-rol — gewone user-PAT's veroorzaken issue #3127 (JWT-gepropageerde auto-groepen worden bij elke poll van alle peers gestript)
- **Cache TTL:** 30s (Squid `external_acl_type ttl=30 negative_ttl=10 concurrency=0`)
- **Endpoint:** `GET /lookup?ip=<overlay_ip>` — vereist de `X-Bridge-Secret`-header (env `LOOKUP_SECRET`; fail-secure: een leeg geheim weigert elk verzoek). Retourneert het volledige groepslidmaatschap van de peer (`status`/`user`/`groups`/`os`) of ERR
- **Health:** `GET /health` — open, geen auth, lekt geen identiteit
- **Enkel verbonden peers** worden gecachet — niet-verbonden peers vallen terug op GUI-gegenereerde regels
- **Fail-open ontwerp:** Wanneer onbereikbaar matchen geen identiteitsgebaseerde ACL's, en valt verkeer door naar niet-identiteitsgebonden URL-filtering. Alle andere beveiligingslagen blijven actief.

## Integratiepunten

| Interface | Richting | Protocol | Details |
|-----------|----------|----------|---------|
| NetBird Management API | Uitgaand (poll) | HTTP (NetBird Docker-netwerk) | `/api/peers` + `/api/users` via `management:80`, service-PAT-auth |
| Squid external_acl | Inkomend (per verzoek) | HTTP (management-LAN) | `192.168.122.23:8088` |
| NATS JetStream | Uitgaand (publish) | NATS (sase-internal Docker-netwerk) | `identity.peer.connected`, `identity.peer.disconnected`, `identity.multi_persona` subjects |

## Bekende problemen / valkuilen

- **Overlay-IP-instabiliteit:** NetBird overlay-IP's zijn niet stabiel bij herinschrijving. Addendum H had `.95.98` hardgecodeerd, wat na herinschrijving `.218.100` werd. Cache-invalidatie moet gebaseerd zijn op peer-ID, niet op IP als primaire sleutel. Zie [Bevinding: Overlay-IP-instabiliteit](../findings/overlay-ip-instability.md).
- **Squid overlay bind-race:** Squid probeert de overlay-listener te binden op `100.x.x.x` voordat NetBird wt0 het overlay-IP heeft toegewezen → errno 49 (adres niet beschikbaar). Zie [Bevinding: Squid overlay bind-race](../findings/squid-overlay-bind-race.md).
- **Service-PAT vereist:** Gewone user-PAT's veroorzaken NetBird issue #3127. Zie [Bevinding: NetBird issue #3127](../findings/netbird-issue-3127.md).

## Gerelateerd

- [Beslissing: NetBird Service PAT](../decisions/netbird-service-pat.md)
- [Component: NetBird](netbird.md)
- [Component: Squid](squid.md)
- [Component: NATS JetStream](nats-jetstream.md)
- [Concept: Identity Flow](../concepts/identity-flow.md)

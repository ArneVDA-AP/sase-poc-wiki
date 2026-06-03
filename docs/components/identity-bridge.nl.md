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

De antwoordstroom: Squid stuurt `<IP> <group>` → helper bevraagt `GET /member?ip=<overlay_ip>` → Identity Bridge retourneert `OK user=<email>` of `ERR` → Squid's persona-ACL wordt geactiveerd of valt door.

## Configuratie

- **Service-gebruiker:** `identity-bridge` met admin-rol — gewone user-PAT's veroorzaken issue #3127 (JWT-gepropageerde auto-groepen worden bij elke poll van alle peers gestript)
- **Cache TTL:** 30s (Squid `external_acl_type ttl=30 negative_ttl=10 children-max=5`)
- **Endpoint:** `GET /member?ip=<overlay_ip>` — retourneert persona-groep of ERR
- **Health:** `GET /health`
- **Enkel verbonden peers** worden gecachet — niet-verbonden peers vallen terug op GUI-gegenereerde regels
- **Fail-open ontwerp:** Wanneer onbereikbaar matchen geen identiteitsgebaseerde ACL's, en valt verkeer door naar niet-identiteitsgebonden URL-filtering. Alle andere beveiligingslagen blijven actief.

## Integratiepunten

| Interface | Richting | Protocol | Details |
|-----------|----------|----------|---------|
| NetBird Management API | Uitgaand (poll) | HTTPS (overlay 100.x.x.x) | `/api/peers` + `/api/users`, PAT-auth |
| Squid external_acl | Inkomend (per verzoek) | HTTP (management-LAN) | `192.168.122.23:8088` |
| NATS JetStream | Uitgaand (publish) | NATS (sase-internal Docker-netwerk) | `identity.login`, `identity.group_change` subjects |

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

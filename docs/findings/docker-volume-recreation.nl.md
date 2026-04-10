---
title: "Bevinding: Docker-volume-mounts vereisen containerrecreatie"
tags: [finding, workaround]
---

# Bevinding: Docker-volume-mounts vereisen containerrecreatie

**Component:** [Caddy](../components/caddy.md), [NetBird](../components/netbird.md)  
**Ernst:** Valkuil

## Wat er gebeurde

Na het toevoegen van een nieuwe volume-mount aan de `caddy`-service in `docker-compose.yml` maakte het uitvoeren van `docker compose restart caddy` de nieuwe mount niet zichtbaar in de container. Het verwachte bestandspad was niet toegankelijk en de service bleef zich gedragen alsof de wijziging niet had plaatsgevonden.

## Oorzaak

`docker compose restart` stuurt SIGTERM dan SIGKILL naar het containerproces en herstart het — maar hergebruikt de bestaande containerconfiguratie, inclusief de volume-mounts die aanwezig waren toen de container voor het eerst werd aangemaakt. Nieuwe volume-mounts in `docker-compose.yml` worden alleen toegepast wanneer de container opnieuw wordt aangemaakt.

## Oplossing / workaround

Gebruik `docker compose up -d <service>` in plaats van `docker compose restart <service>`:

```bash
docker compose up -d caddy
```

`docker compose up -d` detecteert dat de containerconfiguratie is gewijzigd (door de nieuwe volume-mount) en maakt de container opnieuw aan met de bijgewerkte configuratie.

## Lessen

- `docker compose restart` ≠ "configuratiewijzigingen toepassen" — het herstart alleen de bestaande container
- `docker compose up -d` past configuratiewijzigingen toe door containers opnieuw aan te maken waar nodig
- Dit is een veelvoorkomend Docker Compose-misverstand dat aanzienlijke debugtijd verspilt — wanneer een configuratiewijziging niet van kracht lijkt te zijn, probeer altijd `up -d` vóór de aanname dat de configuratie fout is

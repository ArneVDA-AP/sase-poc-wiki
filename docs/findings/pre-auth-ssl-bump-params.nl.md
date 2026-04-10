---
title: "Bevinding: pre-auth ssl-bump-parameters vereist"
tags: [finding, squid, ssl-bump, proxy]
---

# Bevinding: pre-auth ssl-bump-parameters vereist

**Component:** [Squid](../components/squid.md)  
**Ernst:** Blokker

## Wat er gebeurde

Na het toevoegen van een Squid-listener op het NetBird-overlay-IP via pre-auth include (`http_port 100.70.154.79:3128`) werd HTTPS-verkeer van mobile01 via de proxy niet onderschept door ClamAV of de DLP-server. Het verkeer leek door te gaan zonder inspectie.

## Oorzaak

Het pre-auth include-bestand bevatte alleen de kale `http_port`-directive zonder ssl-bump-parameters. Zonder `ssl-bump cert=... generate-host-certificates=on` verwerkt Squid HTTPS als een ondoorzichtige `CONNECT`-tunnel — het stuurt versleutelde bytes door zonder te decrypteren. ICAP-services ontvangen nooit inhoud om te inspecteren.

De OPNsense-GUI voegt ssl-bump-parameters alleen toe aan de door hem gegenereerde listener (het LAN-IP). De GUI heeft geen kennis van het pre-auth include-bestand.

## Oplossing / workaround

Het pre-auth include-bestand moet de volledige ssl-bump-directive bevatten:

```bash
echo 'http_port 100.70.154.79:3128 ssl-bump cert=/var/squid/ssl/ca.pem \
  dynamic_cert_mem_cache_size=10MB generate-host-certificates=on' \
  > /usr/local/etc/squid/pre-auth/netbird-listener.conf
configctl proxy restart
```

## Lessen

- Elke `http_port`-directive die SSL Bump nodig heeft, moet expliciet de ssl-bump-parameters bevatten — er is geen overerving van andere listeners
- Het pre-auth include-bestand is niet zichtbaar in `squid -k parse`-uitvoer (dat toont alleen het hoofdbestand), maar het is wel geladen en actief
- Na elke wijziging van de pre-auth include is `configctl proxy restart` vereist — niet een GUI-toepassen

---
title: "Bevinding: Cosmos hostname op IP beperkt OAuth/SSO"
tags: [finding, cosmos, application-gateway, oidc, mfa, tls]
---

# Bevinding: Cosmos hostname op IP beperkt OAuth/SSO

**Component:** [Cosmos](../components/cosmos.nl.md)
**Ernst:** Gotcha

## Wat er gebeurde

Cosmos werd opgezet met zijn hostname als het IP `10.0.0.100` (gesloten lab, geen FQDN). De MFA-gate per route werkt voor Uptime Kuma op `kuma.dc.local`, maar de `AuthEnabled`-toggle is grijs en kan niet opgeslagen worden voor routes op een `*.dc.local`-hostname (Gitea). Cross-domain SSO tussen de Cosmos admin-origin en de applicatieroutes functioneert niet.

## Oorzaak

Met de hostname op een IP bouwt Cosmos zijn OAuth-redirect-URL als `https://10.0.0.100/cosmos-ui/openid`. Browser-cookies zijn domain-gebonden: een sessie-cookie op `10.0.0.100` is niet geldig op `gitea.dc.local`. Wanneer Cosmos de OAuth/SSO-flow probeert af te ronden voor een route op een andere hostname, wordt de cookie die op de admin-origin (`10.0.0.100`) is gezet nooit teruggepresenteerd op het domein van de route (`*.dc.local`), en kan de auth-binding niet tot stand komen. Cosmos toont dit door de `AuthEnabled`-toggle voor `*.dc.local`-routes uit te schakelen (grijs): de functie is onbeschikbaar, niet enkel ongeconfigureerd.

De onderliggende beperking is architecturaal, geen bug: een OAuth/SSO redirect-en-cookie-flow vereist één parent-domein dat de issuer en de beveiligde routes delen. Een IP-literal kan geen parent van `*.dc.local` zijn, dus de cross-domain-cookie kan nooit geldig zijn.

## Oplossing / workaround

Voor de PoC werd de beperking bewust aanvaard: de kernfuncties van de gateway (identity gate op `kuma.dc.local`, SmartShield, container-isolatie) werken allemaal, en Gitea blijft sowieso bewust zonder de Cosmos MFA-gate (het heeft zijn eigen volwassen auth). Zie [Beslissing: Twee-laags ZTNA](../decisions/cosmos-two-layer-ztna.nl.md).

De echte fix is Cosmos een FQDN onder de gedeelde `dc.local`-parent geven, zodat de OAuth-issuer en de routes onder één domein vallen:

1. Voeg `cosmos.dc.local` toe aan dc01 `/etc/hosts`.
2. Wijzig de hostname in Cosmos **Configuration → General** naar `cosmos.dc.local`.
3. Herstart Cosmos en hervalideer de OAuth-redirect-URL's.

Na migratie wordt de redirect `https://cosmos.dc.local/cosmos-ui/openid`, deelt een cookie op `cosmos.dc.local` de `dc.local`-parent met `gitea.dc.local`/`kuma.dc.local`, en wordt de `AuthEnabled`-toggle bruikbaar op `*.dc.local`-routes. Deze migratie staat genoteerd als open punt en is uitgesteld in de huidige build.

## Lessen

- Voor elke reverse proxy / application gateway die OAuth/SSO over meerdere sub-hostnames doet: zet de eigen hostname van de gateway vanaf dag één op een FQDN onder hetzelfde parent-domein als de routes. Een IP-literal sluit cross-domain-cookies uit.
- Een grijze UI-toggle kan een signaal zijn van een structurele beperking, niet van een ontbrekende instelling. Hier is de grijze `AuthEnabled` Cosmos die terecht een config weigert die nooit een geldige cookie zou kunnen opleveren.
- De beperking en de architectuurkeuze per service staan los van elkaar: Gitea staat by design gate-uit, dus op de parallelle stack blokkeerde deze bevinding niets. Maar ze zou dat wel doen voor een toekomstige `*.dc.local`-service die de Cosmos MFA-gate wél nodig heeft.

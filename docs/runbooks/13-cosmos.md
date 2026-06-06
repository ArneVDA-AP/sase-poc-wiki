---
title: "Runbook: Cosmos Application Gateway"
tags: [runbook, cosmos, application-gateway, mfa, smartshield, docker, dns, netbird]
---

# Runbook: Cosmos Application Gateway

**Source:** `rayan_Cosmos Implementatie SASE PoC.md`, `DEMO_SCRIPT_COSMOS_MFA.md`
**Node(s):** dc01 (`10.0.0.100`) + OPNsense (Unbound, `wt0`) + NetBird Dashboard + Windows client
**Prerequisites:** NetBird overlay operational with `10.0.0.0/8` advertised to peers ([Runbook 02: ZTNA Overlay](02-ztna-overlay.md)); dc01 reachable over the overlay
**Status:** ✅ PoC-validated on the parallel stack — sandbox integration pending

> **Scope.** This runbook targets the parallel `SASE_POC` environment (dc01 `10.0.0.100`), not the main sandbox. Cosmos is an additive application-admission layer on top of NetBird; where it overlaps the sandbox (DNS, the overlay), the sandbox is source of truth. Access to dc01 in this environment is via the jump host: `ssh -J root@10.158.10.67:9022 dc@10.0.0.100`.

---

## Prerequisites checklist

- [ ] dc01 running Ubuntu 22.04+ (build uses 24.04.4 LTS), 2+ vCPU, 3 GB+ RAM, static IP `10.0.0.100/24`, gateway `10.0.0.1` (OPNsense LAN)
- [ ] dc01 has internet via OPNsense + NAT (for Docker image pulls)
- [ ] Ports 80 and 443 free on dc01 — no other webserver may run
- [ ] DNS resolution via Unbound on OPNsense (`10.0.0.1`)
- [ ] NetBird overlay up; dc01 reachable from a NetBird client (Runbook 02)
- [ ] Authenticator app ready (Microsoft / Google Authenticator)

---

## Step 1: DC-1 preparation

Connect to dc01 over the jump host:

```bash
ssh -J root@10.158.10.67:9022 dc@10.0.0.100
```

> **Gotcha: nginx must be masked, not just stopped.** Ubuntu Server may install nginx as a dependency; it grabs ports 80/443 that Cosmos needs, and `systemctl stop` is not enough because nginx restarts after a reboot.

```bash
sudo systemctl stop nginx
sudo systemctl disable nginx
sudo systemctl mask nginx
sudo systemctl status nginx     # must show: masked
```

Install Docker (official repo), then pin the daemon config — both settings below are mandatory for this GNS3 environment:

> **Gotcha: two critical daemon.json settings together.** A **custom address pool** keeps Docker bridges off the GNS3 ranges (`10.0.0.0/8`, `192.168.122.0/24`), and **MTU 1400** leaves room for GNS3/KVM encapsulation overhead. Default MTU 1500 causes packet fragmentation and mysterious connectivity loss.

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

**Verify:**

```bash
sudo docker info | grep -A 3 "Default Address Pools"   # must show 192.168.200.0/20
sudo docker info | grep MTU                            # must show 1400
```

---

## Step 2: Install Cosmos

> **Gotcha: no Portainer / CasaOS / Unraid templates.** The official Cosmos docs are explicit — those break the configuration. Always use the `docker run` command below.

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

`--network host` lets Cosmos bind 80/443 on the host; `--privileged` + the Docker socket let it manage the other containers; `/var/lib/cosmos:/config` is persistent config, certs, and DB credentials.

**Verify (after ~30 s):**

```bash
sudo docker ps | grep cosmos-server
sudo docker logs cosmos-server --tail 20
```

Reach the UI via an SSH tunnel (DNS is not configured yet). In a separate terminal:

```bash
ssh -J root@10.158.10.67:9022 -L 8080:10.0.0.100:443 dc@10.0.0.100
```

Browser: `https://localhost:8080/cosmos-ui/`. The self-signed certificate warning is expected — continue through it.

---

## Step 3: Initial configuration wizard

On first open, Cosmos runs a setup wizard. Set:

| Field | Value | Reason |
|-------|-------|--------|
| Hostname | `10.0.0.100` | Closed lab; see the OAuth caveat below |
| HTTPS Mode | `SELFSIGNED` | No Let's Encrypt in a closed lab |
| Allow insecure via local IP | On | Admin access via IP without a cert warning |
| Force Multi-Factor Authentication | On | Mandatory for all users |
| Admin username / password | `admin` / `<admin-password>` | |

> **Gotcha: hostname on IP limits OAuth/SSO (architectural).** With the hostname as an IP, Cosmos builds its OAuth redirect as `https://10.0.0.100/cosmos-ui/openid`; because cookies are domain-bound, the `AuthEnabled` toggle is greyed for `*.dc.local` routes and cross-domain SSO does not work. This is accepted deliberately in the PoC. To migrate later: add `cosmos.dc.local` to dc01 `/etc/hosts`, change the hostname in Configuration → General, restart Cosmos. See [Finding: hostname on IP limits OAuth](../findings/cosmos-hostname-oauth.md).

After saving and logging in as admin, a **New MFA Setup** screen shows a QR code. Open Microsoft / Google Authenticator → add account → scan → enter the 6-digit token → Login. Keep this app entry — every Cosmos login needs it.

> **Gotcha: Constellation VPN is a paid feature in v0.22.10.** Do not activate it; NetBird ZTNA replaces it (NetBird advertises `10.0.0.0/8` to all peers). And SmartShield has **no global toggle** in v0.22.10 — it is configured per route, automatically when installing via Cosmos Market.

---

## Step 4: DNS configuration

DNS must be set in two places: Unbound on OPNsense (so clients resolve) and `/etc/hosts` on dc01 (so Cosmos itself resolves).

**Unbound host overrides** — OPNsense UI → Services → Unbound DNS → Overrides → Host Overrides → + Add:

| Host | Domain | IP |
|------|--------|----|
| `cosmos` | `dc.local` | `10.0.0.100` |
| `kuma` | `dc.local` | `10.0.0.100` |
| `gitea` | `dc.local` | `10.0.0.100` |

Save → Apply Changes.

> **Gotcha: Cosmos' internal resolver does not know `dc.local`.** The single most surprising pitfall. Cosmos runs as a Docker container and uses Docker's internal resolver (`127.0.0.11`), which does not know `dc.local`. A route on `kuma.dc.local` fails with `lookup kuma.dc.local: no such host` until the name is added to dc01's own hosts file (the container inherits it).

```bash
sudo tee -a /etc/hosts <<'EOF'
10.0.0.100 gitea.dc.local
10.0.0.100 kuma.dc.local
10.0.0.100 cosmos.dc.local
EOF
cat /etc/hosts | grep dc.local   # must show all three
```

Add every new `*.dc.local` service to this file as you deploy it, or its route returns an internal server error.

---

## Step 5: NetBird DNS integration

NetBird-connected clients must resolve `*.dc.local`. This needs a NetBird nameserver rule and an OPNsense firewall rule.

**NetBird nameserver rule** — NetBird management UI → DNS → Nameservers → Add: Nameserver IP `10.0.0.1`, Match domains `dc.local`, Groups `All`. This tells NetBird to send `dc.local` queries to Unbound on OPNsense.

> **Gotcha: OPNsense blocks DNS from the NetBird overlay by default.** NetBird traffic arrives on the `wt0` interface; OPNsense blocks port 53 from non-LAN interfaces by default, so a firewall rule is required.

OPNsense UI → Firewall → Rules → WireGuard (`wt0`) → + Add: Action Pass, Protocol TCP/UDP, Source any, Destination This Firewall, Destination Port 53. Save → Apply.

> **Gotcha: the school search domain hijacks resolution.** The AP Hogeschool network pushes a `bletchley.cloud` search domain via DHCP, which makes Unbound append it (`kuma.dc.local.bletchley.cloud`) and resolve to an external IP. Fix: OPNsense → Services → Unbound DNS → General → check the "Domain" field → remove `bletchley.cloud` if present → Apply.

**Verify (Windows client, NetBird connected):**

```powershell
netbird status                 # Status: Connected
nslookup kuma.dc.local         # must resolve to 10.0.0.100
ping 10.0.0.100                # must reply
```

---

## Step 6: Services — Uptime Kuma (with MFA gate)

Uptime Kuma is the monitoring dashboard; we put it behind the Cosmos MFA gate.

Cosmos UI → App Market → search "Uptime Kuma" → Install. Set **Hostname** `kuma.dc.local`; leave the rest default.

> **Gotcha: do not install the socket-proxy.** Cosmos Market may offer a "socket-proxy" alongside Uptime Kuma — it causes a link error and is not needed. Deactivate that option if offered.

Cosmos generates the route security config automatically (`SmartShield` on, `AuthEnabled` true, `BlockCommonBots`, `ThrottlePerMinute` 12000, `cosmos-force-network-secured` true). If the route still shows `10.0.0.100` after install: Cosmos UI → URLs → Uptime-Kuma → Edit → Use Host On, Hostname `kuma.dc.local` → Save.

**Verify (Windows client, NetBird connected):** browse to `https://kuma.dc.local`. Expected flow: Cosmos login (`admin` / `<admin-password>`) → MFA code from Authenticator → Kuma's own login → dashboard. Test with an **incognito window** to prove the gate — an active 48h session would otherwise skip the MFA screen (this is session management, not a bypass).

### Gitea (without MFA gate)

Gitea is deployed deliberately **without** the Cosmos MFA gate — it has its own mature auth, so a second MFA layer would be redundant (see [Decision: Two-layer ZTNA](../decisions/cosmos-two-layer-ztna.md)). Cosmos UI → App Market → "Gitea" → Install, Hostname `gitea.dc.local`. In Gitea's first-run wizard, set **SSH Server Port** to `222` (Docker binds Gitea SSH on 222) and leave all database/SQLite paths untouched.

> **Gotcha: Gitea port 3000 is not externally bound.** Gitea's `3000/tcp` is intentionally not exposed — it is reachable only through Cosmos. For an Uptime Kuma monitor, use `https://10.0.0.100` (Cosmos), never `:3000`.

---

## Step 7: CA certificate (optional, for a clean demo)

Cosmos serves a self-signed cert; browsers warn until the CA is trusted. To remove the warning on a client, serve `cosmos-ca.crt` from dc01 and import it:

```bash
# On dc01: temporary HTTP server in the directory holding cosmos-ca.crt
cd /home/dc && python3 -m http.server 8888
```

```powershell
# On the Windows client (NetBird connected):
Invoke-WebRequest -Uri http://10.0.0.100:8888/cosmos-ca.crt -OutFile C:\cosmos-ca.crt
certutil -addstore -f "Root" C:\cosmos-ca.crt
```

Stop the dc01 HTTP server (`Ctrl+C`), fully close and reopen the browser, browse to `https://kuma.dc.local` — the padlock should be green with no warning.

---

## Step 8: MFA enrolment verification (demo gate)

Prove the gate end to end from a clean state:

1. Open an incognito/private window (no active Cosmos session).
2. Browse to `https://kuma.dc.local` → Cosmos login page appears (not Kuma).
3. Enter `admin` / `<admin-password>` → Login → Cosmos MFA screen appears (Kuma still not visible).
4. Enter the 6-digit code from Authenticator → Cosmos redirects to the Uptime Kuma dashboard.

This is the two-gate model in action: NetBird admitted the device, Cosmos admits the user with MFA.

---

## Final verification

- [ ] `sudo docker ps` shows `cosmos-server` (host 80+443), `cosmos-mongo-*`, `Uptime-Kuma` (healthy), `Gitea`
- [ ] `sudo docker info` confirms address pool `192.168.200.0/20` and MTU 1400
- [ ] nginx shows `masked`
- [ ] `nslookup kuma.dc.local` from a NetBird client resolves to `10.0.0.100`
- [ ] Incognito → `https://kuma.dc.local` shows the Cosmos login, then MFA, then Kuma
- [ ] `https://gitea.dc.local` reaches Gitea (no Cosmos MFA gate, by design)
- [ ] (Optional) CA imported on the client; no "Not secure" warning
- [ ] Uptime Kuma monitors use IP addresses (e.g. `https://10.0.0.100`), not `*.dc.local`

---

## Related

- [Component: Cosmos](../components/cosmos.md)
- [Concept: Application Gateway](../concepts/application-gateway.md)
- [Concept: Zero Trust](../concepts/zero-trust.md)
- [Decision: Two-layer ZTNA (NetBird + Cosmos)](../decisions/cosmos-two-layer-ztna.md)
- [Finding: Cosmos hostname on IP limits OAuth/SSO](../findings/cosmos-hostname-oauth.md)
- [Component: NetBird](../components/netbird.md)
- [Runbook 02: ZTNA Overlay](02-ztna-overlay.md)

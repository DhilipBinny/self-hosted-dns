# OWN_DNS — Self-Hosted Ad-Blocking DNS Server

**Host:** <xx_server_ip_xx> (Ubuntu 24.04, Docker Swarm, Portainer)
**Domain:** <xx_domain_xx> (DDNS)
**Date:** 2026-02-18

---

## Architecture Overview

```
                         INTERNET
                            |
               ┌────────────┴────────────┐
               |                         |
          iPhone (DoH)            Android (DoT)
          HTTPS :443              TLS :853
               |                         |
               └────────────┬────────────┘
                            |
              TP-Link HB710 Router (NAT)
              <xx_domain_xx> → Public IP
              Port forwards: 80, 443, 853 → <xx_server_ip_xx>
              DHCP DNS: <xx_server_ip_xx> (fallback: 1.1.1.1)
                            |
        ════════════════════╪═══════════════════════
        LAN <xx_lan_subnet_xx>/24 |  Host: <xx_server_ip_xx>
        ════════════════════╪═══════════════════════
                            |
           ┌────────────────┼────────────────┐
           |                |                |
      Port 443         Port 853         Port 53
      (DoH/HTTPS)      (DoT/TLS)       (Plain DNS)
           |                |                |
    ┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐
    |   Nginx     |  |   Nginx     |  |  AdGuard    |
    |  HTTP block |  | Stream block|  |   Home      |
    |  GeoIP2     |  |  GeoIP2     |  |  (direct)   |
    |  Country    |  |  Country    |  |  LAN only   |
    |  filter     |  |  filter     |  |             |
    |             |  |             |  |             |
    | SSL termina-|  | TLS pass-   |  |             |
    | tion + proxy|  | through     |  |             |
    └──────┬──────┘  └──────┬──────┘  └──────┘──────┘
           |                |                |
     HTTP :8080       TLS :8530              |
     (unencrypted     (AdGuard               |
      DoH proxy)       handles TLS)          |
           |                |                |
           └────────────────┼────────────────┘
                            |
                   ┌────────┴────────┐
                   |   AdGuard Home  |
                   |                 |
                   | DNS :53         |
                   | Dashboard :8080 |
                   | DoT :853→8530   |
                   |                 |
                   | Ad/tracker      |
                   | blocking        |
                   | 19 blocklists   |
                   | DNS cache (4MB) |
                   └────────┬────────┘
                            |
                   Upstream DNS (encrypted)
                   ┌────────┴────────┐
                   |                 |
              Cloudflare         Quad9
         dns.cloudflare.com  dns10.quad9.net
            (DoH)               (DoH)
```

### Data Flow — DoH (iPhone)

```
iPhone → HTTPS :443 → Router NAT → Nginx :443
  → GeoIP check (IN/SG only) → SSL terminate
  → HTTP proxy to AdGuard :8080/dns-query
  → AdGuard resolves (with ad blocking)
  → Response back through Nginx → iPhone
```

### Data Flow — DoT (Android)

```
Android → TLS :853 → Router NAT → Nginx :853 (host) → :8530 (container)
  → GeoIP check (IN/SG only) → TLS passthrough (NO termination)
  → Forward raw TLS to AdGuard :8530 (host) → :853 (container)
  → AdGuard terminates TLS, resolves DNS (with ad blocking)
  → Response back through Nginx → Android
```

### Data Flow — Plain DNS (LAN)

```
LAN device → DNS :53 → AdGuard Home directly
  → Ad blocking + resolve → Response
```

### Port Mapping Summary

```
External    Host          Container     Service         Protocol
───────     ────          ─────────     ───────         ────────
:53    →    :53      →    :53           AdGuard         Plain DNS (TCP+UDP)
:443   →    :443     →    :443          Nginx HTTP      DoH (HTTPS)
:853   →    :853     →    :8530         Nginx Stream    DoT (TLS passthrough)
:8080  →    :8080    →    :80           AdGuard         Dashboard (HTTP)
 n/a        :8530    →    :853          AdGuard         DoT internal (TLS)
```

**Key design:** Nginx listens on host :853 (container :8530) and passes TLS
through to AdGuard on host :8530 (container :853). AdGuard handles its own
TLS termination. This avoids double-TLS and lets AdGuard see DoT clients.

---

## Remote Machine: <xx_server_ip_xx>

**SSH:** `ssh <xx_user_xx>@<xx_server_ip_xx>`

### Host System Changes

| File | Change | Backup |
|------|--------|--------|
| `/etc/systemd/resolved.conf` | `DNSStubListener=no`, `DNS=127.0.0.1` | `/etc/systemd/resolved.conf.bak` |
| `/etc/resolv.conf` | Symlink → `/run/systemd/resolve/resolv.conf` | Was → `/run/systemd/resolve/stub-resolv.conf` |

**Why:** Ubuntu's systemd-resolved occupies port 53 by default. Disabling its stub listener frees port 53 for AdGuard.

---

## Folder Structure on Remote

```
<xx_data_path_xx>/
├── adguard/
│   ├── conf/
│   │   └── AdGuardHome.yaml          # AdGuard main config
│   └── work/
│       ├── data/                      # Runtime data (query logs, stats)
│       └── adguard-doh.mobileconfig   # iOS DoH profile (for download)
│
├── ssl/                               # Let's Encrypt certificates
│   ├── live/<xx_domain_xx>/
│   │   ├── fullchain.pem             # SSL certificate (symlink)
│   │   └── privkey.pem               # SSL private key (symlink)
│   ├── archive/<xx_domain_xx>/       # Actual cert files
│   ├── renewal/<xx_domain_xx>.conf   # Certbot renewal config
│   ├── renewal-hooks/                # Pre/post/deploy hooks
│   └── accounts/                     # Let's Encrypt account
│
├── nginx-config/
│   └── nginx.conf                     # Nginx GeoIP config (HTTP + Stream)
│
├── nginx-build/
│   └── Dockerfile                     # Builds nginx-geoip2:1.29.0 image
│
├── nginx-html/
│   └── index.html                     # Public landing page (status page)
│
├── geoip2/
│   ├── GeoLite2-Country.mmdb         # MaxMind country database
│   └── .geoipupdate.lock             # Update lock file
│
├── certbot-webroot/                   # Certbot HTTP challenge directory
│
└── .env                               # Backup credentials (chmod 600, not used by services)
```

---

## Docker Stack (Portainer)

**Stack name:** `dns-adguard`
**Mode:** Docker Swarm

### Stack YAML (deployed via Portainer)

```yaml
version: "3.8"

services:
  adguard:
    image: adguard/adguardhome:latest
    ports:
      - target: 53
        published: 53
        protocol: tcp
        mode: host
      - target: 53
        published: 53
        protocol: udp
        mode: host
      - target: 80
        published: 8080
        protocol: tcp
        mode: host
      - target: 853
        published: 8530
        protocol: tcp
        mode: host
    volumes:
      - <xx_data_path_xx>/adguard/work:/opt/adguardhome/work
      - <xx_data_path_xx>/adguard/conf:/opt/adguardhome/conf
      - <xx_data_path_xx>/ssl:/etc/letsencrypt:ro
    deploy:
      replicas: 1
      restart_policy:
        condition: any

  nginx-geoip:
    image: nginx-geoip2:1.29.0
    ports:
      - target: 443
        published: 443
        protocol: tcp
        mode: host
      - target: 8530
        published: 853
        protocol: tcp
        mode: host
    volumes:
      - <xx_data_path_xx>/nginx-config/nginx.conf:/etc/nginx/nginx.conf:ro
      - <xx_data_path_xx>/nginx-html:/usr/share/nginx/html:ro
      - <xx_data_path_xx>/geoip2:/usr/share/GeoIP:ro
      - <xx_data_path_xx>/ssl:/etc/letsencrypt:ro
    deploy:
      replicas: 1
      restart_policy:
        condition: any

  geoip-updater:
    image: maxmindinc/geoipupdate:latest
    environment:
      - GEOIPUPDATE_ACCOUNT_ID=${GEOIP_ACCOUNT_ID}
      - GEOIPUPDATE_LICENSE_KEY=${GEOIP_LICENSE_KEY}
      - GEOIPUPDATE_EDITION_IDS=GeoLite2-Country
      - GEOIPUPDATE_FREQUENCY=72
      - GEOIPUPDATE_VERBOSE=1
    volumes:
      - <xx_data_path_xx>/geoip2:/usr/share/GeoIP
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure

  certbot-renew:
    image: certbot/certbot
    volumes:
      - <xx_data_path_xx>/ssl:/etc/letsencrypt
      - <xx_data_path_xx>/certbot-webroot:/var/www/html
    entrypoint: /bin/sh -c "while true; do certbot renew --webroot -w /var/www/html; sleep 12h; done"
    deploy:
      replicas: 1
      restart_policy:
        condition: any
```

### Services Summary

| Service | Image | Purpose | Host Ports |
|---------|-------|---------|------------|
| **adguard** | `adguard/adguardhome:latest` | DNS server + ad blocking + DoT TLS termination + dashboard | 53/tcp, 53/udp, 8080, 8530 |
| **nginx-geoip** | `nginx-geoip2:1.29.0` (custom) | GeoIP country filter for DoH (SSL termination) and DoT (TLS passthrough) | 443, 853 |
| **geoip-updater** | `maxmindinc/geoipupdate:latest` | Auto-updates GeoLite2 country database every 72h | None |
| **certbot-renew** | `certbot/certbot:latest` | Auto-renews Let's Encrypt SSL cert (checks every 12h) | None |

### Docker Images on Host

| Image | How obtained |
|-------|-------------|
| `adguard/adguardhome:latest` | `docker pull` |
| `nginx-geoip2:1.29.0` | Built from `<xx_data_path_xx>/nginx-build/Dockerfile` (includes `sub_filter` module) |
| `maxmindinc/geoipupdate:latest` | `docker pull` |
| `certbot/certbot:latest` | `docker pull` |

### Credentials & Secrets

| Secret | Where stored | Notes |
|--------|-------------|-------|
| MaxMind Account ID | Portainer stack env var `GEOIP_ACCOUNT_ID` | Passed via `${GEOIP_ACCOUNT_ID}` substitution |
| MaxMind License Key | Portainer stack env var `GEOIP_LICENSE_KEY` | Passed via `${GEOIP_LICENSE_KEY}` substitution |
| AdGuard admin password | AdGuard internal (bcrypt in `AdGuardHome.yaml`) | Login at `/adguard/` |
| Backup `.env` file | `<xx_data_path_xx>/.env` (chmod 600) | Reference copy, not used by services |

**No credentials are hardcoded in any config file or stack YAML.**

### Web Pages

| URL | Access | Content |
|-----|--------|---------|
| `https://<xx_domain_xx>/` | Public (GeoIP) | Status landing page — DoH/DoT status, GeoIP, response time |
| `https://<xx_domain_xx>/adguard/` | GeoIP + AdGuard login | AdGuard Home dashboard (proxied via `sub_filter` rewriting) |
| `https://<xx_domain_xx>/dns-query` | Public (GeoIP) | DoH endpoint for iPhone/Android |

### AdGuard Custom Filtering Rules (Whitelist)

```
@@||updates.maxmind.com^
```

Required so the geoip-updater container can reach MaxMind's servers (blocked by default by some blocklists).

---

## Config Files

### Nginx GeoIP Config (`nginx-config/nginx.conf`)

```nginx
load_module /usr/lib/nginx/modules/ngx_http_geoip2_module.so;
load_module /usr/lib/nginx/modules/ngx_stream_geoip2_module.so;

user  nginx;
worker_processes  auto;
error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format main '$geoip2_data_country_code -- $status -- '
                    '$remote_addr [$time_local] "$request" '
                    '$body_bytes_sent "$http_user_agent"';
    access_log /var/log/nginx/access.log main;

    # GeoIP2 Country Database
    geoip2 /usr/share/GeoIP/GeoLite2-Country.mmdb {
        auto_reload 60m;
        $geoip2_data_country_code country iso_code;
    }

    # Allow only India and Singapore
    map $geoip2_data_country_code $allowed_country {
        default 0;
        IN 1;
        SG 1;
    }

    # Private IP whitelist (always allow LAN)
    geo $private_ip {
        default 0;
        <xx_lan_subnet_xx>/24 1;
        192.168.1.0/24 1;
        10.0.0.0/8 1;
        172.16.0.0/12 1;
        127.0.0.0/8 1;
    }

    # Combined logic: allow if country OK or private IP
    map "$allowed_country:$private_ip" $block_request {
        default 1;
        "1:0" 0;
        "1:1" 0;
        "0:1" 0;
    }

    # DoH reverse proxy (port 443)
    server {
        listen 443 ssl http2;
        server_name <xx_domain_xx>;

        ssl_certificate /etc/letsencrypt/live/<xx_domain_xx>/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/<xx_domain_xx>/privkey.pem;

        # Block non-allowed countries
        if ($block_request) {
            return 403;
        }

        # DoH endpoint — proxy to AdGuard HTTP (unencrypted DoH)
        location /dns-query {
            proxy_pass http://<xx_server_ip_xx>:8080/dns-query;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_pass_request_body on;
            proxy_set_header Content-Type $content_type;
            proxy_set_header Accept $http_accept;
        }

        # AdGuard WebUI — GeoIP protected, AdGuard handles its own login
        location = /adguard {
            return 301 /adguard/;
        }

        location /adguard/ {
            proxy_pass http://<xx_server_ip_xx>:8080/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_redirect / /adguard/;

            # Rewrite absolute paths in HTML/JS responses (requires sub_filter module)
            sub_filter_once off;
            sub_filter_types text/html text/css text/javascript application/javascript application/json;
            sub_filter 'href="/' 'href="/adguard/';
            sub_filter 'src="/' 'src="/adguard/';
            # ... (additional sub_filter rules for /control/, /login, /static/, etc.)
        }

        # Public landing page
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
    }
}

# DoT stream proxy (port 853) with GeoIP blocking
stream {
    # GeoIP2 Country Database for stream
    geoip2 /usr/share/GeoIP/GeoLite2-Country.mmdb {
        auto_reload 60m;
        $geoip2_stream_country_code country iso_code;
    }

    # Allow only India and Singapore
    map $geoip2_stream_country_code $stream_allowed_country {
        default 0;
        IN 1;
        SG 1;
    }

    # Private IP whitelist
    geo $stream_private_ip {
        default 0;
        <xx_lan_subnet_xx>/24 1;
        192.168.1.0/24 1;
        10.0.0.0/8 1;
        172.16.0.0/12 1;
        127.0.0.0/8 1;
    }

    # Route allowed traffic to AdGuard, blocked traffic to nowhere
    map "$stream_allowed_country:$stream_private_ip" $dot_backend {
        default 127.0.0.1:1;       # Blocked — connection refused
        "1:0" <xx_server_ip_xx>:8530;  # Allowed country
        "1:1" <xx_server_ip_xx>:8530;  # Allowed country + private IP
        "0:1" <xx_server_ip_xx>:8530;  # Private IP (any country)
    }

    log_format stream_main '$remote_addr [$time_local] '
                           'country=$geoip2_stream_country_code '
                           'upstream=$dot_backend '
                           'status=$status';
    access_log /var/log/nginx/stream_access.log stream_main;

    # TLS passthrough — AdGuard handles TLS termination
    server {
        listen 8530;
        proxy_pass $dot_backend;
    }
}
```

**Key points:**
- `http` block: Nginx terminates SSL, checks GeoIP, proxies plain HTTP to AdGuard's unencrypted DoH endpoint (:8080)
- `stream` block: Nginx does NOT terminate TLS — it passes raw TLS through to AdGuard (:8530→:853) which handles its own TLS. This is "TLS passthrough"
- Both blocks have independent GeoIP country checking (IN, SG only)
- Blocked DoH requests get HTTP 403; blocked DoT connections are sent to `127.0.0.1:1` (connection refused)

### Nginx GeoIP Dockerfile (`nginx-build/Dockerfile`)

Multi-stage build that compiles Nginx 1.29.0 from source with:
- `ngx_http_geoip2_module` (dynamic module for HTTP GeoIP)
- `ngx_stream_geoip2_module` (dynamic module for Stream/DoT GeoIP)
- `--with-stream` (TCP/UDP proxying for DoT)
- `--with-http_ssl_module` (HTTPS)
- `--with-http_v2_module` (HTTP/2)
- `--with-http_realip_module` (real client IP behind proxy)
- `--with-http_sub_module` (response body rewriting for `/adguard/` subpath proxy)

Base: `debian:bookworm-slim`

### AdGuard Home Config (key settings in `AdGuardHome.yaml`)

| Setting | Value |
|---------|-------|
| Admin user | `admin` |
| DNS listen | `0.0.0.0:53` |
| Dashboard | `0.0.0.0:80` (mapped to 8080 on host) |
| DoT listen | `0.0.0.0:853` (mapped to 8530 on host) |
| Upstream DNS | `https://dns10.quad9.net/dns-query`, `https://dns.cloudflare.com/dns-query` |
| Bootstrap DNS | `1.1.1.1`, `8.8.8.8`, `9.9.9.10`, `149.112.112.10` |
| `allow_unencrypted_doh` | `true` (allows Nginx to proxy DoH over HTTP) |
| TLS cert | `/etc/letsencrypt/live/<xx_domain_xx>/fullchain.pem` |
| TLS key | `/etc/letsencrypt/live/<xx_domain_xx>/privkey.pem` |
| Cache | Enabled, 4MB |
| DNSSEC | Disabled |
| Rate limit | 20 req/s |

### AdGuard Blocklists (19 active)

| Category | Lists |
|----------|-------|
| **Core ad blocking** | AdGuard DNS filter, OISD Big, HaGeZi Pro, 1Hosts Xtra |
| **Annoyances** | AdGuard Popup Hosts, Anti Push Notifications |
| **IoT / Smart TV** | Smart-TV Blocklist |
| **Security** | Threat Intelligence Feeds, Phishing Army, Phishing URL Blocklist, URLHaus, Stalkerware Indicators, Anti-Malware List, Badware Hoster Blocklist |
| **Privacy** | URL Shortener, Apple Tracker (iCloud Private Relay) |
| **Content** | HaGeZi Gambling, HaGeZi Anti-Piracy, ShadowWhisperer Dating |

Managed via AdGuard dashboard: Filters > DNS Blocklists.

---

## Router Config (TP-Link HB710)

| Setting | Value |
|---------|-------|
| DHCP Primary DNS | `<xx_server_ip_xx>` |
| DHCP Secondary DNS | `1.1.1.1` (fallback) |
| Port forward 80 | → `<xx_server_ip_xx>:80` |
| Port forward 443 | → `<xx_server_ip_xx>:443` |
| Port forward 853 | → `<xx_server_ip_xx>:853` |

---

## Client Setup

### LAN devices (automatic)
Router DHCP hands out `<xx_server_ip_xx>` as DNS. No per-device config needed.
All LAN traffic uses plain DNS (:53) directly to AdGuard — no GeoIP filtering needed.

### iPhone (anywhere via DoH)
1. Download `adguard-doh.mobileconfig` onto iPhone
2. Settings > General > VPN & Device Management > AdGuard Home DNS > Install
3. Uses DoH: `https://<xx_domain_xx>/dns-query`
4. Traffic flow: iPhone → :443 → Nginx (GeoIP + SSL) → AdGuard :8080

### Android (anywhere via DoT)
1. Settings > Network & Internet > Private DNS
2. Select "Private DNS provider hostname"
3. Enter: `<xx_domain_xx>`
4. Traffic flow: Android → :853 → Nginx (GeoIP + TLS passthrough) → AdGuard :8530

---

## GeoIP Country Blocking

| Country | Code | Access |
|---------|------|--------|
| India | IN | Allowed |
| Singapore | SG | Allowed |
| Private IPs | 192.168.x.x, 10.x.x.x, 172.16-31.x.x | Allowed |
| All others | * | Blocked (DoH: HTTP 403, DoT: connection refused) |

**MaxMind Account:** See `.env` file on remote
**Database:** GeoLite2-Country (auto-updated every 72h by geoip-updater service)

GeoIP is applied at both layers:
- **DoH (HTTP):** `ngx_http_geoip2_module` — returns HTTP 403 for blocked countries
- **DoT (Stream):** `ngx_stream_geoip2_module` — routes blocked countries to `127.0.0.1:1` (connection refused)

---

## SSL Certificate

| Field | Value |
|-------|-------|
| Domain | `<xx_domain_xx>` |
| Issuer | Let's Encrypt |
| Renewal | Auto (certbot-renew service checks every 12h) |
| Path | `<xx_data_path_xx>/ssl/live/<xx_domain_xx>/` |

Used by:
- **Nginx** for DoH SSL termination (port 443)
- **AdGuard** for DoT TLS termination (port 853→8530)

### Initial Certificate Setup

Before deploying the stack, you need to obtain the first SSL certificate. This must be done **before** Nginx starts (it will fail without certs).

**Prerequisites:**
- Port 80 must be forwarded from your router to `<xx_server_ip_xx>:80`
- Your DDNS domain must resolve to your public IP
- No other service should be using port 80

```bash
# 1. Create the SSL directory
mkdir -p <xx_data_path_xx>/ssl

# 2. Obtain the initial certificate (standalone mode — runs its own temp web server on :80)
docker run --rm -p 80:80 \
  -v <xx_data_path_xx>/ssl:/etc/letsencrypt \
  certbot/certbot certonly \
  --standalone \
  -d <xx_domain_xx> \
  --agree-tos \
  --no-eff-email \
  --register-unsafely-without-email

# 3. Verify the cert was created
ls <xx_data_path_xx>/ssl/live/<xx_domain_xx>/
# Should show: fullchain.pem  privkey.pem  cert.pem  chain.pem
```

**After first cert is obtained:**
- Deploy the full stack (Nginx, AdGuard, certbot-renew, etc.)
- The `certbot-renew` service handles all future renewals automatically using webroot mode
- Nginx serves the ACME challenge at `/.well-known/acme-challenge/` for renewals
- Certs are checked every 12h, renewed when < 30 days remain

**If cert expires or gets corrupted:**
```bash
# Stop the stack first (to free port 80)
docker stack rm dns-adguard

# Re-issue cert using standalone mode
docker run --rm -p 80:80 \
  -v <xx_data_path_xx>/ssl:/etc/letsencrypt \
  certbot/certbot certonly \
  --standalone \
  -d <xx_domain_xx> \
  --agree-tos \
  --no-eff-email \
  --register-unsafely-without-email

# Redeploy the stack
# (via Portainer UI or docker stack deploy)
```

---

## Useful Commands

```bash
# SSH to remote
ssh <xx_user_xx>@<xx_server_ip_xx>

# Check all services
docker service ls | grep dns-adguard

# Check service logs
docker service logs dns-adguard_adguard
docker service logs dns-adguard_nginx-geoip

# Check Nginx GeoIP access log (DoH)
docker exec $(docker ps -q --filter name=nginx-geoip) cat /var/log/nginx/access.log

# Check Nginx stream log (DoT)
docker exec $(docker ps -q --filter name=nginx-geoip) cat /var/log/nginx/stream_access.log

# Restart a service
docker service update --force dns-adguard_adguard
docker service update --force dns-adguard_nginx-geoip

# Test plain DNS
dig @<xx_server_ip_xx> google.com +short

# Test ad blocking
dig @<xx_server_ip_xx> ads.google.com +short
# Should return 0.0.0.0 (blocked)

# Test DoH
curl -s -o /dev/null -w '%{http_code}' \
  -H 'content-type: application/dns-message' \
  -H 'accept: application/dns-message' \
  --data-binary @<(printf '\x00\x00\x01\x00\x00\x01\x00\x00\x00\x00\x00\x00\x06google\x03com\x00\x00\x01\x00\x01') \
  'https://<xx_domain_xx>/dns-query'
# Should return 200

# Test DoT
dig @<xx_domain_xx> google.com +short +tls

# Check GeoIP logs (country codes)
docker logs $(docker ps -q --filter name=nginx-geoip) 2>&1 | tail -20

# Rebuild nginx-geoip2 image (if needed)
cd <xx_data_path_xx>/nginx-build && docker build -t nginx-geoip2:1.29.0 .

# Manually renew SSL cert
docker run --rm -p 80:80 \
  -v <xx_data_path_xx>/ssl:/etc/letsencrypt \
  certbot/certbot certonly --standalone -d <xx_domain_xx> \
  --agree-tos --no-eff-email --register-unsafely-without-email
```

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Port 53 in use on host | systemd-resolved stub listener | Set `DNSStubListener=no` in `/etc/systemd/resolved.conf` |
| DoH returns "Not Found" | `allow_unencrypted_doh` is false | Edit `AdGuardHome.yaml`, set `allow_unencrypted_doh: true`, restart |
| DoH returns 400 | Nginx not passing request body | Add `proxy_pass_request_body on` + Content-Type/Accept headers |
| DoT timeout (loop) | Nginx :853 proxying to host :853 (itself) | Map AdGuard 853→8530, Nginx proxies to :8530 |
| DoT "end of file" | Double TLS (Nginx SSL + AdGuard TLS) | Remove `ssl` from Nginx stream listener — use TLS passthrough |
| Network sandbox join failed | Stale Docker overlay network | `docker stack rm`, wait, `docker network prune -f`, redeploy |
| Swarm deploy fails | `container_name` / `restart: unless-stopped` | Use `deploy.restart_policy.condition` instead, remove container_name |
| GeoIP updater connection refused | AdGuard blocking `updates.maxmind.com` | Whitelist: `@@\|\|updates.maxmind.com^` in Filters > Custom filtering rules |

---

## Full Revert (undo everything)

```bash
# 1. Remove the stack
docker stack rm dns-adguard

# 2. Remove project files
rm -rf <xx_data_path_xx>

# 3. Remove custom docker image
docker rmi nginx-geoip2:1.29.0

# 4. Restore systemd-resolved
sudo cp /etc/systemd/resolved.conf.bak /etc/systemd/resolved.conf
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
sudo systemctl restart systemd-resolved

# 5. Revert router
# TP-Link HB710: Advanced > Network > Internet > Advanced
# Change DNS back to Automatic
# Remove port forwards for 80, 443, 853
```

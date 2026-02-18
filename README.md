# Self-Hosted DNS

Self-hosted ad-blocking DNS server with AdGuard Home, Nginx GeoIP2 country filtering, DoH and DoT support.

## Architecture

```
Internet
   |
   +-- iPhone (DoH :443) ──┐
   +-- Android (DoT :853) ─┤
   |                        |
   |              Router (NAT + DDNS)
   |              Port forwards: 443, 853, 80
   |                        |
   |              ┌─────────┴─────────┐
   |              |   Nginx GeoIP2    |
   |              |   Country filter  |
   |              |   (IN / SG only)  |
   |              └─────────┬─────────┘
   |                        |
   |              ┌─────────┴─────────┐
   +-- LAN (:53)──|   AdGuard Home    |
                  |   Ad blocking     |
                  |   19 blocklists   |
                  └─────────┬─────────┘
                            |
                  Upstream DNS (encrypted)
                  Cloudflare + Quad9 (DoH)
```

## Features

- **Ad blocking** — blocks ads, trackers, malware, phishing across all devices
- **DNS-over-HTTPS (DoH)** — encrypted DNS for iPhone (port 443)
- **DNS-over-TLS (DoT)** — encrypted DNS for Android (port 853)
- **GeoIP country filtering** — only allows connections from whitelisted countries
- **LAN DNS** — plain DNS on port 53 for all local devices via router DHCP
- **Auto-updating GeoIP** — MaxMind GeoLite2 database refreshed every 72h
- **Auto-renewing SSL** — Let's Encrypt cert renewed automatically via certbot
- **AdGuard WebUI** — accessible at `/adguard/` behind GeoIP protection

## Stack

| Service | Image | Purpose |
|---------|-------|---------|
| AdGuard Home | `adguard/adguardhome` | DNS server, ad blocking, DoT termination |
| Nginx GeoIP2 | `nginx-geoip2:1.29.0` (custom) | GeoIP filtering, DoH SSL termination, DoT passthrough |
| GeoIP Updater | `maxmindinc/geoipupdate` | Auto-updates country database |
| Certbot | `certbot/certbot` | Auto-renews SSL certificates |

## Prerequisites

- Linux server (tested on Ubuntu 24.04)
- Docker Swarm + Portainer (or plain Docker)
- A DDNS domain pointing to your public IP
- Router with port forwarding (80, 443, 853)
- MaxMind GeoLite2 account ([free signup](https://www.maxmind.com/en/geolite2/signup))

## Quick Start

1. **Clone and configure**
   ```bash
   git clone https://github.com/DhilipBinny/self-hosted-dns.git
   cd self-hosted-dns
   ```

2. **Replace placeholders** — see [PLACEHOLDERS.txt](PLACEHOLDERS.txt) for the full list
   ```
   <xx_domain_xx>        Your DDNS domain
   <xx_server_ip_xx>     Your server LAN IP
   <xx_lan_subnet_xx>    Your LAN subnet
   <xx_data_path_xx>     Data directory on host
   ```

3. **Set up credentials**
   ```bash
   cp .env.example .env
   # Edit .env with your MaxMind account ID and license key
   ```

4. **Obtain SSL certificate**
   ```bash
   mkdir -p /path/to/OWN_DNS/ssl
   docker run --rm -p 80:80 \
     -v /path/to/OWN_DNS/ssl:/etc/letsencrypt \
     certbot/certbot certonly --standalone \
     -d your.domain.net \
     --agree-tos --no-eff-email --register-unsafely-without-email
   ```

5. **Build the custom Nginx image**
   ```bash
   cd nginx-geoip
   docker build -t nginx-geoip2:1.29.0 .
   ```

6. **Deploy**
   ```bash
   docker stack deploy -c docker-compose.yml dns-adguard
   ```
   Or deploy via Portainer UI with env vars from `.env`.

7. **Configure your router**
   - DHCP DNS: your server IP
   - Port forward 80, 443, 853 to your server

## Client Setup

**iPhone (DoH)**
- Install the `adguard-doh.mobileconfig` profile
- Settings > General > VPN & Device Management > Install

**Android (DoT)**
- Settings > Network > Private DNS > enter your domain

**LAN devices**
- Automatic via router DHCP — no config needed

## Project Structure

```
├── nginx-geoip/
│   ├── Dockerfile              # Custom Nginx build (GeoIP2 + sub_filter + stream)
│   └── nginx.conf              # Nginx config (DoH, DoT, /adguard/ proxy)
├── docker-compose.yml          # Portainer Swarm stack
├── adguard-doh.mobileconfig    # iOS DoH profile
├── .env.example                # Credential template
├── PLACEHOLDERS.txt            # Placeholder reference
└── SETUP.md                    # Full setup documentation
```

## Documentation

See [SETUP.md](SETUP.md) for detailed documentation covering:
- Complete architecture and data flow diagrams
- Port mapping reference
- All config files with explanations
- AdGuard blocklist details
- SSL certificate management
- Troubleshooting guide
- Full revert instructions

## License

[MIT](LICENSE)

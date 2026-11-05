# Plexible

A solution for accessing Plex Media Server as a local network client when proxying through reverse proxies, eliminating remote access restrictions without requiring Plex Pass.

## Table of Contents

- [Problem Statement](#problem-statement)
- [How Plex Determines Local vs Remote](#how-plex-determines-local-vs-remote)
- [Solution Architecture](#solution-architecture)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Troubleshooting](#troubleshooting)
- [Technical Details](#technical-details)
- [License](#license)

## Problem Statement

Plex Media Server has different behavior for "local" vs "remote" clients:
- Local clients can play content without Plex Pass restrictions
- Remote clients may require Plex Pass for playback outside home network
- Direct play/streaming permissions and quality restrictions apply differently

When using a reverse proxy (Caddy, Nginx, etc.), Plex sees the proxy's IP address as the request source, causing it to misclassify the connection as remote even if it's coming from your local network or through your own infrastructure.

## How Plex Determines Local vs Remote

Plex uses multiple signals to classify connections:

1. **Source IP Address**: Compared against `allowedNetworks` in Preferences.xml and server's interface subnets
2. **Forwarded Headers**: Plex parses `X-Forwarded-For`, `X-Real-IP`, and similar headers to determine the original client IP
3. **Host Header Validation**: Requests must come from recognized hosts in `customConnections`
4. **Network Topology**: Plex checks if the source IP is in the same subnet as the server's interfaces
5. **WAN Detection**: Combined with `TreatWanIPAsLocal` setting

The key insight is that Plex trusts forwarded headers over the socket source IP in many cases, and also classifies based on whether the source IP belongs to a local network subnet.

## Solution Architecture

```
┌───────────────────────┐     ┌───────────────────────┐     ┌───────────────────────┐     ┌───────────────────────┐
│         Client        │────>│         Caddy         │────>│      Local Proxy      │────>│          Plex         │
│                       │     │    (Reverse Proxy)    │     │      (Container)      │     │     (Media Server)    │
└───────────────────────┘     └───────────────────────┘     └───────────────────────┘     └───────────────────────┘
            │                             │                             │                             │
 HTTPS request                      Strip headers              Connect from LAN IP             Sees LAN source
                                   Rewrite headers                                           Classifies as local
```

### Components

1. **Reverse Proxy (Caddy)**: Terminates TLS, routes requests, strips identifying headers
2. **Local Proxy (Container)**: Makes connections to Plex from a LAN IP address, ensuring proper source IP presentation
3. **Network Translation (nftables/iptables)**: Optional redirection rules for traffic routing
4. **Plex Media Server**: Configured to recognize local network ranges

## Prerequisites

- Docker and Docker Compose
- Plex Media Server
- A reverse proxy setup
- Network access to modify routing/firewall rules (for advanced scenarios)

## Installation

### 1. Clone and Setup

```bash
git clone https://github.com/bezko/plexible.git
cd plexible
cp .env.example .env
```

### 2. Configure Environment

Edit `.env` with your settings:
```env
PUID=1000
PGID=1000
TZ=UTC
MEDIA_PATH=./media
LAN_IP=192.168.1.100
ADVERTISE_IP=http://192.168.1.100:32400,https://your-proxy-hostname
```

### 3. Configure Caddy

Edit `Caddyfile` with your configuration. The setup routes through a local proxy container to ensure proper source IP handling.

### 4. Configure Plex

Access Plex and go to Settings > Network:
- **Secure connections**: Preferred
- **Allowed Networks**: Add your LAN subnets (e.g., `192.168.0.0/16,10.0.0.0/8,172.16.0.0/12,127.0.0.1`)
- **Custom server access URLs**: Add your proxy URLs
- **Treat WAN IP As Local**: Enable as needed

### 5. Start Services

```bash
docker compose up -d
```

## Configuration

### Header Stripping (Critical)

Plex prioritizes forwarded headers. Strip these headers in your reverse proxy:

- `X-Forwarded-For` - Contains original client IP
- `X-Forwarded-Host` - Original host
- `X-Forwarded-Proto` - Original protocol
- `X-Real-IP` - Alternative forwarded IP header
- `X-Plex-Forwarded-For` - Plex-specific forwarded header
- `Forwarded` - RFC 7239 forwarded header
- `Referer` and `Origin` - Can leak proxy information

### Host Header

Set `Host` to match a local IP address that Plex recognizes as local. This prevents Plex from treating the request as coming through a foreign host.

### Location Header Rewriting

Plex often sends redirects with internal/LAN IPs. Rewrite these to use your proxy hostname.

### Local Proxy Container

The local proxy container connects to Plex from a LAN IP address on the same network, ensuring Plex sees requests originating from a local subnet rather than from localhost or container networks.

## Troubleshooting

### Check Plex Classification

Monitor Plex logs to see how it classifies requests:
```bash
docker logs plex | grep -E 'Request.*\(WAN\)|Request.*\(Subnet\)|Request.*\(Loopback\)'
```

- `(Loopback)` or `(Subnet)` = Local (good)
- `(WAN)` = Remote (bad, may trigger restrictions)

### Common Issues

**Still classified as WAN:**
- Check X-Forwarded-For headers are being stripped
- Verify source IP is in allowedNetworks  
- Ensure requests come from LAN subnet IP
- Check customConnections includes your proxy URL

**"Not hosted by Plex" warning:**
- Add proxy hostname to customConnections
- Set Secure Connections appropriately
- Ensure Host header matches recognized values

**Redirects go to unexpected IPs:**
- Use header_down to rewrite Location headers in your proxy

## Technical Details

### Why This Works

1. **Source IP Control**: Using a local proxy container on the LAN network ensures Plex sees requests from LAN IP addresses
2. **Header Stripping**: Removing forwarded headers prevents Plex from inferring the original remote client IP
3. **Subnet Matching**: When the source IP is in the same subnet as Plex's interfaces and included in allowedNetworks, Plex classifies as local
4. **Host Recognition**: Proper configuration of customConnections and Host headers satisfies Plex's validation

### Network Architecture

The solution uses a dedicated local proxy container that:
- Runs on the host/LAN network
- Accepts connections from the reverse proxy
- Forwards requests to Plex using a LAN source IP
- Strips or rewrites headers as needed

## License

This project is released under [The Unlicense](LICENSE).

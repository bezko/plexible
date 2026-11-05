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
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Client     │────▶│    Caddy     │────▶│  Local Proxy │────▶│     Plex     │
│              │     │ (Reverse     │     │  (IP Spoof)  │     │  (host net)  │
└─────────────┘     │  Proxy)      │     └─────────────┘     └─────────────┘
                    └─────────────┘            │                     │
                           │                    │                     │
                        Strips headers    Source appears as    Sees LAN IP
                        Sets Host to LAN   LAN IP subnet        Classifies as local
```

### Components

1. **Reverse Proxy (Caddy)**: Terminates TLS, routes requests, strips identifying headers
2. **Network Translation (nftables/iptables)**: Redirects traffic so Plex sees LAN source IPs
3. **Header Manipulation**: Removes forwarded headers Plex uses for remote detection
4. **Plex Configuration**: Properly configured networks and connections

## Prerequisites

- Docker and Docker Compose
- Plex Media Server
- A reverse proxy (Caddy included in compose)
- Network access to modify routing/firewall rules (for DNAT approach)

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
TZ=America/New_York
MEDIA_PATH=/path/to/your/media
ADVERTISE_IP=http://192.168.1.100:32400,https://plex.yourdomain.com
```

### 3. Configure Caddy

Edit `Caddyfile` to set your hostname and ensure proper header manipulation:

```caddyfile
your-hostname {
    tls internal
    reverse_proxy 127.0.0.1:32400 {
        header_up Host 127.0.0.1:32400
        header_up -X-Forwarded-For
        header_up -X-Forwarded-Host
        header_up -X-Forwarded-Proto
        header_up -Referer
        header_up -Origin
        header_up -X-Real-IP
        header_down -Location
    }
}
```

### 4. Configure Plex

Access Plex at `http://localhost:32400/web` and go to Settings > Network:

- **Secure connections**: Preferred (or as needed)
- **Allowed Networks**: Add your LAN subnets, e.g., `192.168.0.0/16,10.0.0.0/8,172.16.0.0/12,127.0.0.1`
- **Custom server access URLs**: Add your proxy URLs
- **Treat WAN IP As Local**: Enable if needed

### 5. Start Services

```bash
docker compose up -d
```

## Configuration

### Header Stripping (Critical)

Plex prioritizes forwarded headers. You MUST strip these:

- `X-Forwarded-For` - Contains original client IP
- `X-Forwarded-Host` - Original host
- `X-Forwarded-Proto` - Original protocol
- `X-Real-IP` - Alternative forwarded IP header
- `X-Plex-Forwarded-For` - Plex-specific forwarded header
- `Forwarded` - RFC 7239 forwarded header
- `Referer` and `Origin` - Can leak proxy information

### Host Header

Set `Host` to match a local IP that Plex recognizes as local, or ensure it matches entries in customConnections.

### Location Header Rewriting

Plex often sends redirects with LAN IPs. Rewrite these to use your proxy hostname:

```caddyfile
header_down Location https://your-hostname/web/index.html
```

### Network Translation (Advanced)

For cases where traffic comes in on specific interfaces, use nftables:

```nft
table ip plex_local {
    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;
        iifname "your-interface" ip saddr your.subnet/24 tcp dport 32400 dnat to 127.0.0.1:PORT
    }
}
```

Load with `nft -f nftables-plex.conf`

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
- Ensure Plex sees requests from LAN subnet IP
- Check customConnections includes your proxy URL

**"Not hosted by Plex" warning:**
- Add proxy hostname to customConnections
- Set Secure Connections appropriately
- Ensure Host header is properly configured

**Redirects go to LAN IP:**
- Use header_down to rewrite Location headers
- Update customConnections

## Technical Details

### Why This Works

1. **Source IP Spoofing via DNAT**: By routing traffic through a local proxy that connects to Plex from 127.0.0.1 or a LAN IP, Plex sees a local source IP
2. **Header Stripping**: Removing forwarded headers prevents Plex from using them to determine the original remote client IP
3. **Subnet Matching**: If source IP is in same subnet as Plex's interfaces and in allowedNetworks, Plex classifies as local
4. **Host Recognition**: Proper customConnections configuration prevents security warnings

### Network Modes

- **host**: Simplest, Plex uses host networking directly
- **bridge**: More isolated but requires additional port mapping and may complicate DNAT

This solution works best with host networking or careful bridge configuration with proper routing.

## License

This project is released under [The Unlicense](LICENSE).

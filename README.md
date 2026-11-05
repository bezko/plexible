# Plexible

A solution for accessing Plex Media Server as a local network client when proxying through reverse proxies, eliminating remote access restrictions without requiring Plex Pass.

## Problem Statement

Plex determines whether a client is "local" or "remote" based on:
1. Source IP address vs configured allowed networks
2. Server's own network interface subnets  
3. Host header validation against custom connections
4. Forwarded headers (X-Forwarded-For) that Plex uses to determine the original client IP

When proxying through services like Caddy, Tailscale, or other reverse proxies, Plex sees the proxy's IP address as the source, causing it to classify connections as remote and requiring Plex Pass for remote playback.

## Solution

This project implements a multi-layer approach to make Plex see connections as genuinely local:

### 1. Network-Level Source IP Spoofing

For traffic coming through reverse proxies, we need Plex to see a source IP from its local subnet. This is achieved using network address translation:

- **DNAT (Destination NAT)**: Intercept incoming connections on the proxy interface/ports and redirect them through a local proxy that makes requests to Plex from a LAN IP address
- **SNAT/Forwarding**: Ensure proper routing so Plex sees the connection originating from a LAN IP rather than localhost or proxy IPs

### 2. Header Manipulation

Plex's request parser has specific behaviors:
- Plex prioritizes `X-Forwarded-For` header over the actual socket source IP for determining client location
- Plex validates the `Host` header against configured connections and will log warnings if unrecognized
- Certain headers like `Referer`, `Origin`, and `X-Plex-Forwarded-For` can influence Plex's behavior

The solution strips problematic forwarding headers and sets appropriate Host headers.

### 3. Plex Configuration

Proper Plex configuration is essential:
- `allowedNetworks`: Must include all proxy/LAN subnets (e.g., `192.168.0.0/16`, `10.0.0.0/8`, `172.16.0.0/12`)
- `customConnections`: Must include the proxy's public-facing hostname/URL for Plex to recognize it
- `TreatWanIPAsLocal`: Can help in certain cases
- `ADVERTISE_IP`: Should include all URLs clients may use to reach the server

### 4. Caddy Reverse Proxy Configuration

The reverse proxy must:
- Forward the original Host header appropriately, or rewrite it
- Strip X-Forwarded-For and related headers that Plex uses to determine remote status
- Rewrite Location headers from Plex to use the proxy's hostname
- Proxy to Plex via a LAN IP address so the source appears local

## Architecture

```
Client -> Caddy (HTTPS) -> LAN IP:32400 (Plex)
                ↓
            DNAT rules redirect traffic
            Headers stripped/rewritten
            Source IP appears as LAN
```

## Implementation Details

### Docker Compose Setup

The provided `docker-compose.yml` demonstrates the complete setup with:
- Plex Media Server in host or bridge network mode
- Caddy as reverse proxy
- Proper networking configuration
- Volume mounts for configuration

### nftables/iptables Rules

For host networking or DNAT scenarios, packet manipulation rules ensure Plex sees LAN source IPs.

### Caddyfile

Configures the reverse proxy with header manipulation and proper routing.

## Usage

1. Copy `.env.example` to `.env` and configure your network settings
2. Update `Caddyfile` with your domain/hostname
3. Configure Plex's network settings via its web interface
4. Run `docker compose up -d`
5. Access Plex through your proxy - it should be treated as local

## Important Notes

- This solution is for personal use on networks you own
- Plex's terms of service should be respected
- The approach manipulates how Plex perceives connections; behavior may change with Plex updates
- Always test in a safe environment first

## License

This project is released under The Unlicense. See LICENSE file for details.

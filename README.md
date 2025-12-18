Media server stack — VPN-backed downloaders (Option 2: Proxy)

Overview
--------
This stack routes downloader outbound traffic through a VPN while keeping ARR
apps (Radarr, Sonarr, Prowlarr, Lidarr) and frontends reachable on your
LAN/internal network. Key pieces:

- `gluetun` — establishes the VPN tunnel (OpenVPN/NordVPN supported).
- `vpn-proxy` — a small SOCKS5 proxy that runs inside `gluetun`'s network
	namespace via `network_mode: service:gluetun`. The proxy listens on
	`gluetun:1080` for SOCKS5 connections and its outbound traffic exits
	through the VPN.
- Downloaders & ARR apps — run on the `media_net` bridge and talk to each
	other using Docker DNS (e.g., `qbittorrent:8080`). Downloaders should be
	configured to use the SOCKS5 proxy for actual downloads.

Prerequisites
-------------
- NordVPN account (or another VPN provider supported by `gluetun`).
	- Username and password (or provider-specific token) are required.
	- Some providers may require OpenVPN config files; check `gluetun` docs.
- If you use Usenet: a Usenet provider account (e.g., Newshosting) and NNTP
	credentials for NZBGet.
- Host requirements:
	- Docker Engine and Docker Compose (or Docker Desktop with Compose v2).
	- A Linux host with a TUN device (`/dev/net/tun`) is required for the VPN
		container to create the VPN tunnel. Docker Desktop on Windows does not
		reliably provide a `/dev/net/tun` device — run on a Linux VM/host or
		WSL2 configured to expose `/dev/net/tun` (advanced).
	- Sufficient disk space and permissions for your `CONFIG_ROOT` and `MEDIA_ROOT`.

Files in this repo
------------------
- `docker-compose.yml` — services, networks, healthchecks, and logging.
 - `.env.template` — template configuration file; copy to `.env` and edit (`.env` is gitignored). Your VPN credentials (OPENVPN_USER, OPENVPN_PASSWORD) should also be added to this .env file.

Security and environment variable handling
-----------------------------------------
- Do NOT commit a populated `.env` to source control. Ensure it's listed in `.gitignore`.
- VPN credentials (`OPENVPN_USER` and `OPENVPN_PASSWORD`) are loaded from your `.env` file by the `gluetun` service.


Preparing configuration
-----------------------
1. Copy `.env.template` to `.env` and edit the values for your host (the
	repository now ships a safe template and `.env` is gitignored):

```bash
cp .env.template .env
# Open and edit .env in your editor
nano .env
# or on Windows: notepad .env
```


3. Ensure the host has `/dev/net/tun` available. On a Linux host run:

```bash
ls -l /dev/net/tun
```

If it does not exist, the VPN container cannot start.

Starting the stack
------------------

```bash
docker compose up -d
```

Verification
------------
- Watch gluetun logs for VPN status:

```bash
docker compose logs -f gluetun
```

- From a downloader container, test the SOCKS proxy (example using curl):

```bash
docker compose exec qbittorrent curl --socks5-hostname gluetun:1080 -sSf https://ifconfig.co
```

	The returned IP should be your VPN exit IP, not your ISP IP.

- Confirm ARR apps can reach downloaders using service names:

```bash
docker compose exec prowlarr curl -fsS http://qbittorrent:8080 || echo "cannot reach qbittorrent"
```

App-specific configuration notes
--------------------------------
- qBittorrent:
	- In Settings -> Connection, set the SOCKS5 proxy to host `gluetun`, port `1080`.
	- Test torrents after enabling the proxy to ensure traffic exits via the VPN.

- NZBGet:
	- In Settings -> Network/Proxy, set SOCKS proxy to `gluetun:1080`.

- Radarr/Sonarr/Prowlarr:
	- Use service hostnames to reach downloaders (e.g., `qbittorrent:8080`).
	- Set indexer HTTP calls to use `HTTP_PROXY`/`HTTPS_PROXY` if needed.

Cloudflare Tunnel (optional) — publish Jellyfin/Jellyseerr securely
----------------------------------------------------------------
Recommended: run Cloudflare Tunnel (`cloudflared`) on the host (or a container)
that can reach the `media_net` services. Example `cloudflared` ingress snippet:

```yaml
tunnel: <TUNNEL-UUID>
credentials-file: /path/to/credentials.json
ingress:
	- hostname: jellyfin.example.com
		service: http://localhost:8096
	- hostname: jellyseerr.example.com
		service: http://localhost:5055
	- service: http_status:404

# Run cloudflared on the same host and map the Docker network or run cloudflared
# as a container attached to `media_net` so it can resolve container hostnames.
```

Platform notes (Windows users)
-----------------------------
- Docker Desktop on Windows does not provide a TUN device to containers by default.
	The `gluetun` container requires `/dev/net/tun` which is a Linux kernel device.
	Options:
	- Run this stack on a Linux host or VM (recommended).
	- Use WSL2 with a fully configured Linux distro that allows `/dev/net/tun`.
	- Alternatively, run a VPN on the host and do not run `gluetun` in a container.

Troubleshooting
---------------
- If downloads use your real IP: ensure the downloader is configured to use SOCKS5
	(env vars alone do not force all protocols through the proxy).
- If `gluetun` fails with TUN errors: verify `/dev/net/tun` exists and host supports it.
- If environment variable handling fails for VPN credentials: ensure your `.env` file contains `OPENVPN_USER` and `OPENVPN_PASSWORD`.
Next steps I can take for you
----------------------------
- Add explicit per-app configuration snippets for qBittorrent and NZBGet.


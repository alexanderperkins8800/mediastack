# Project: Media Server Stack (VPN-backed downloaders)

## Project Overview

This project provides a Docker Compose stack designed to deploy a comprehensive media server environment with a strong emphasis on privacy for download clients. It achieves this by routing all download client traffic through a VPN using the `gluetun` container and an internal SOCKS5 proxy (`vpn-proxy`). Simultaneously, it ensures that "Arr" applications (Radarr, Sonarr, Prowlarr, Lidarr, Readarr) and frontend media servers (Jellyfin, Jellyseerr) can communicate internally and remain accessible from the local network.

## Main Technologies

*   **Containerization:** Docker, Docker Compose
*   **VPN Management:** `gluetun` (VPN client, supporting various providers like NordVPN)
*   **Internal Proxy:** `go-socks5-proxy` (used by `vpn-proxy` service)
*   **Download Clients:** `qbittorrent` (torrent), `nzbget` (Usenet)
*   **Media Management ("Arr" Suite):**
    *   `Radarr` (Movies)
    *   `Sonarr` (TV Shows)
    *   `Prowlarr` (Indexer management for Arr apps)
    *   `Lidarr` (Music)
    *   `Readarr` (Ebooks and Audiobooks)
*   **Media Servers/Frontends:**
    *   `Jellyfin` (Media server for streaming)
    *   `Jellyseerr` (Media request management)
*   **Container Images:** Primarily LinuxServer.io Docker images for consistency.

## Architecture

The stack operates on a custom `media_net` bridge network, facilitating secure inter-container communication.

1.  **VPN Tunnel:** The `gluetun` container establishes a VPN tunnel to a chosen provider (e.g., NordVPN).
2.  **Internal Proxy:** A `vpn-proxy` service runs within `gluetun`'s network namespace. This ensures all traffic originating from `vpn-proxy` exits through the VPN tunnel. The proxy listens internally on `gluetun:1080` for SOCKS5 connections.
3.  **Download & Arr Apps:** Download clients (`qbittorrent`, `nzbget`) and all "Arr" applications are configured to use `socks5://gluetun:1080` for their outbound internet traffic (e.g., connecting to indexers, download sources), thus routing all critical traffic through the VPN. They reside on the `media_net` for internal communication.
4.  **Frontend Services:** `Jellyfin` and `Jellyseerr` are also on the `media_net` and have their web UIs directly exposed to the host network via port mappings, making them accessible from your local network.

## Building and Running

### Prerequisites

*   **Docker Engine and Docker Compose** (or Docker Desktop with Compose v2) installed on your host system.
*   **Linux host with a TUN device** (`/dev/net/tun`) available, which is essential for the `gluetun` VPN container.
*   **VPN Account details** (username, password) for a provider supported by `gluetun`.
*   **Sufficient disk space** for your `CONFIG_ROOT` (application configurations) and `MEDIA_ROOT` (all your media files).

### Preparation

1.  **Environment Configuration:**
    *   Copy the provided `.env.template` file to `.env` in the project root:
        ```bash
        cp .env.template .env
        ```
    *   Open the newly created `.env` file and edit the placeholder values. Crucially, set:
        *   `PUID`: Your user ID on the host system.
        *   `PGID`: Your group ID on the host system.
        *   `TIMEZONE`: Your local timezone (e.g., `America/New_York`).
        *   `CONFIG_ROOT`: The absolute path on your host where application configuration data will be stored (e.g., `/opt/mediastack/config`).
        *   `MEDIA_ROOT`: The absolute path on your host where all your media files (movies, TV shows, music, audiobooks) are located (e.g., `/mnt/sda1/media/MediaStorage`).
        *   `OPENVPN_USER`: Your VPN service username.
        *   `OPENVPN_PASSWORD`: Your VPN service password.

2.  **Verify TUN device:** Ensure your Linux host has `/dev/net/tun` available:
    ```bash
    ls -l /dev/net/tun
    ```
    If this device does not exist, the `gluetun` VPN container will not be able to start.

### Starting the Stack

Navigate to the project root directory in your terminal and run:

```bash
docker compose up -d
```

### Accessing Web UIs (from host machine)

Once the stack is deployed, you can access the various application web interfaces from your web browser using your server's IP address (or `localhost` if running locally) and the following ports:

*   **Jellyfin:** `http://your-server-ip:8096`
*   **Jellyseerr:** `http://your-server-ip:5055`
*   **qBittorrent:** `http://your-server-ip:9001`
*   **NZBGet:** `http://your-server-ip:6789`
*   **Prowlarr:** `http://your-server-ip:9696`
*   **Radarr:** `http://your-server-ip:7878`
*   **Sonarr:** `http://your-server-ip:8989`
*   **Lidarr:** `http://your-server-ip:8686`
*   **Readarr:** `http://your-server-ip:8787`

## Development Conventions

*   **Environment Variables:** All stack-wide configuration parameters (like user IDs, root paths, and VPN credentials) are managed through a `.env` file, which is based on `.env.template` and is excluded from version control via `.gitignore`.
*   **Sensitive Data:** VPN credentials (`OPENVPN_USER`, `OPENVPN_PASSWORD`) are loaded directly from the `.env` file into the `gluetun` service's environment.
*   **Volume Mappings:** Application-specific configurations are stored in host-mounted volumes under `${CONFIG_ROOT}`, and all media content is managed via a single `${MEDIA_ROOT}` mount point.
*   **Logging:** All services utilize a unified `json-file` logging driver with defined size limits for log rotation.
*   **Healthchecks:** Comprehensive `healthcheck` configurations are implemented for each service to monitor their operational status and responsiveness.

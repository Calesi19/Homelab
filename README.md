# Homelab

Docker Compose stack running on a Raspberry Pi 5 (16 GB): a VPN-routed
\*arr media stack, Plex, and a few dashboards — all private to the Tailscale
tailnet, nothing exposed to the public internet.

## Services

| Service      | Port | Network      | Notes                                    |
|--------------|------|--------------|------------------------------------------|
| Gluetun      | —    | VPN (Nord)   | WireGuard, kill-switched gateway         |
| qBittorrent  | 8080 | via Gluetun  | torrent client                           |
| Sonarr       | 8989 | via Gluetun  | TV automation                            |
| Radarr       | 7878 | via Gluetun  | movie automation                         |
| Prowlarr     | 9696 | via Gluetun  | indexer manager                          |
| Bazarr       | 6767 | via Gluetun  | subtitle automation                      |
| Unpackerr    | 5656 | via Gluetun  | extracts packed releases                 |
| Plex         | 32400| host         | media server (`/web`)                    |
| Overseerr    | 5055 | host         | media request UI                         |
| Tautulli     | 8181 | host         | Plex stats/monitoring                    |
| Homepage     | 3000 | Tailscale-only | dashboard                              |
| FileBrowser  | 8082 | Tailscale-only | browse host files (Quantum fork)       |
| Uptime Kuma  | 3001 | Tailscale-only | service monitoring                     |
| Beszel       | 8090 | Tailscale-only | system monitoring (hub + agent)        |

Media lives at `/mnt/immich_data/media/{tv,movies,downloads}` on an external
SSD. Container configs persist under `./config/` (git-ignored).

## Host setup (fresh Pi)

Tested on Ubuntu 26.04 LTS, Docker 29.x, Compose v5.x.

1. **Mount the data SSD.** Partition, `mkfs.ext4 -L immich_data`, then add to
   `/etc/fstab` (adjust the UUID):
   ```
   UUID=<ssd-uuid> /mnt/immich_data ext4 defaults,nofail 0 2
   ```
   Then `sudo mkdir -p /mnt/immich_data && sudo mount -a`. The `nofail` flag
   keeps the Pi bootable if the drive is unplugged.
2. **Install Tailscale** ([tailscale.com/download](https://tailscale.com/download)),
   `sudo tailscale up`, note the host's Tailscale IP for `TAILSCALE_IP`.
3. **Install Docker**: `curl -fsSL https://get.docker.com | sh`, then
   `sudo usermod -aG docker $USER` and re-login.
4. **Clone this repo** somewhere on the SSD (e.g.
   `/mnt/immich_data/docker/arr-stack`), `cp .env.example .env`,
   `chmod 600 .env`, and fill in every value (see comments in the file).

## First start

```bash
docker compose up -d
docker compose ps
docker logs gluetun --tail 20   # look for "VPN is up"
```

Grab the Plex claim token right before the first start — it expires in
~4 minutes.

Web UIs are at `http://<TAILSCALE_IP>:<port>` (see table above).

## Wiring services together

- In Sonarr/Radarr, the download client is qBittorrent at `127.0.0.1:8080`
  (they share Gluetun's network namespace, so localhost works).
- In Prowlarr, add Sonarr at `http://localhost:8989` and Radarr at
  `http://localhost:7878` — indexers then sync automatically.
- In Overseerr, Sonarr is `http://gluetun:8989` and Radarr is
  `http://gluetun:7878` (Overseerr is *not* behind the VPN, so it reaches
  them through Gluetun's published ports).
- In Bazarr, Sonarr/Radarr are at `localhost:8989` / `localhost:7878`.
- Unpackerr watches `/downloads` and extracts packed releases in place.

## Post-install checklist

- qBittorrent: change the default login immediately; set share limits
  (pause at ratio 1.0) so seeding doesn't fill the disk.
- Sonarr/Radarr: enable "Remove Completed Downloads" on the qBittorrent
  download client so finished downloads clean up after import.
- Uptime Kuma: open the UI once to create the admin account, then add monitors.
- Beszel: create the admin account, Add System, put the TOKEN/KEY in `.env`,
  `docker compose up -d beszel-agent`.
- Homepage: needs `HOMEPAGE_ALLOWED_HOSTS=<TAILSCALE_IP>:3000` (already set).
  Widget API keys go in `config/homepage/services.yaml` (chmod 600).
- FileBrowser: admin user is `admin`; the password is `FILEBROWSER_ADMIN_PASSWORD`.

## Notes

- Everything behind Gluetun is kill-switched: if the VPN drops, those
  containers lose internet until it reconnects. Nothing leaks.
- NordVPN doesn't support port forwarding, so qBittorrent won't be
  connectable — downloads still work, just with fewer peer connections.
- All ports are reachable only on the tailnet/LAN. Nothing is publicly exposed.
- Plex does software transcoding only on the Pi 5 — direct play works great,
  heavy transcoding doesn't.
- Publish only the LAN address in Plex's server settings; never list the
  Tailscale IP there (plex.tv rejects the published-address update while a
  Tailscale IP is listed, which breaks remote/app discovery).

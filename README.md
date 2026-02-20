# Homelab: Nextcloud AIO + Jellyfin + Samba via tsbridge (Tailscale)

This repository documents how to deploy a private homelab using Docker with:

- **[Nextcloud All-in-One (AIO)](https://github.com/nextcloud/all-in-one)** for cloud storage and collaboration
- **[Jellyfin](https://jellyfin.org/docs/general/installation/container/)** for media streaming
- **[Samba](https://github.com/dockur/samba)** for network file access (Windows/macOS/Linux)
- **[tsbridge](https://github.com/jtdowney/tsbridge)** to securely expose services over **Tailscale**
- No public ports and no reverse proxy on the host

All external access is restricted to devices on your Tailnet.

---

## Prerequisites

- Linux server (Tested on [Ubuntu 24.04.3 LTS](https://ubuntu.com/download/server))
- [Docker](https://docs.docker.com/engine/install/ubuntu/) + Docker Compose installed
- A [Tailscale](https://tailscale.com/) account
- A Tailscale OAuth client with tag permissions

---

## Directory Layout

### Configuration files

```
~/homelab
├── docker-compose.yml
├── tsbridge.toml
└── smb.conf
```

### Shared media directories

```
/media/media-library      # External hard drive
└── Movies
└── Shows

/srv/laptop-storage       # Internal laptop drive
```

---

## Step 1 — Install Docker

First, add the Docker apt repository:

```bash
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

Then install Docker:

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker
```

Verify:

```bash
docker version
docker compose version
docker run hello-world
```

---

## Step 2 — Create the Media Directories

```bash
sudo mkdir -p /media/media-library
sudo chown -R 1000:1000 /media/media-library
```

---

## Step 3 — Create the Homelab Directory

```bash
mkdir -p ~/homelab
cd ~/homelab
```

---

## Step 4 — Create the Nextcloud AIO Master Volume

Nextcloud AIO **requires** this exact volume name:

```bash
docker volume create nextcloud_aio_mastercontainer
```

Do **not** rename it. Do **not** let Docker Compose auto-prefix it.

---

## Step 5 — Create the Nextcloud AIO Network

> ⚠️ **This step is critical.** tsbridge connects to the `nextcloud-aio` network to reach
> `nextcloud-aio-apache` by hostname. The compose file declares this network as `external: true`,
> so it **must exist before you run `docker compose up`** — otherwise the entire stack will refuse
> to start with "network not found".

```bash
docker network create nextcloud-aio
```

---

## Step 6 — Configure Tailscale ACLs

In **Tailscale Admin → Access Controls**, ensure your tag is defined:

```json
{
  "tagOwners": {
    "tag:tsbridge": ["YOUR_TAILSCALE_LOGIN_EMAIL"]
  }
}
```

Then generate an OAuth client at **Tailscale Admin → Settings → OAuth clients → Generate OAuth client**. Give it **Devices: Read & Write** scope and assign the `tag:tsbridge` tag. Save the Client ID and Client Secret.

---

## Step 7 — Create `tsbridge.toml`

```bash
nano ~/homelab/tsbridge.toml
```

```toml
[tailscale]
oauth_client_id     = "YOUR_OAUTH_CLIENT_ID"
oauth_client_secret = "YOUR_OAUTH_CLIENT_SECRET"
state_dir           = "/var/lib/tsbridge"
default_tags        = ["tag:tsbridge"]

[[services]]
name         = "jellyfin"
backend_addr = "http://jellyfin:8096"
tags         = ["tag:tsbridge"]

[[services]]
name         = "nextcloud"
backend_addr = "http://nextcloud-aio-apache:11000"
tags         = ["tag:tsbridge"]
```

---

## Step 8 — Create `docker-compose.yml`

```bash
nano ~/homelab/docker-compose.yml
```

```yaml
services:
  tsbridge:
    image: ghcr.io/jtdowney/tsbridge:latest
    container_name: tsbridge
    command: ["-config", "/config/tsbridge.toml"]
    cap_add:
      - NET_ADMIN     # Required: Tailscale needs to manage network interfaces
      - SYS_MODULE    # Required: load kernel modules if needed
    devices:
      - /dev/net/tun:/dev/net/tun   # Required: Tailscale TUN device
    volumes:
      - tsbridge-state:/var/lib/tsbridge
      - /home/YOUR_USERNAME/homelab/tsbridge.toml:/config/tsbridge.toml:ro
    restart: unless-stopped
    networks:
      - default
      - nextcloud-aio   # Must be on this network to reach nextcloud-aio-apache
    depends_on:
      - nextcloud-aio-mastercontainer
      - jellyfin

  nextcloud-aio-mastercontainer:
    image: nextcloud/all-in-one:latest
    container_name: nextcloud-aio-mastercontainer
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - nextcloud_aio_mastercontainer:/mnt/docker-aio-config
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      APACHE_PORT: "11000"
      APACHE_IP_BINDING: "0.0.0.0"
      SKIP_DOMAIN_VALIDATION: "true"

  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    volumes:
      - /media/media-library:/media
      - /srv/laptop-storage:/laptop-storage
      - jellyfin_config:/config
      - jellyfin_cache:/cache
    networks:
      - default

  samba:
    image: dockurr/samba:latest
    container_name: samba
    restart: unless-stopped
    ports:
      - "445:445"
    volumes:
      - /media/media-library:/medialibrary:rw
      - ./smb.conf:/etc/samba/smb.conf:ro
    environment:
      USER: "SAMBA_USERNAME"    # Replace with your desired username
      PASS: "SAMBA_PASSWORD"    # Replace with your desired password
      UID: "1000"
      GID: "1000"

volumes:
  nextcloud_aio_mastercontainer:
    external: true
    name: nextcloud_aio_mastercontainer
  tsbridge-state:
  jellyfin_config:
  jellyfin_cache:

networks:
  nextcloud-aio:
    external: true
    name: nextcloud-aio
```

---

## Step 9 — Start the Stack

```bash
cd ~/homelab
docker compose up -d
```

Verify ports are listening:

```bash
sudo ss -tulpn | grep :8080
sudo ss -tulpn | grep :445
```

Check tsbridge joined Tailscale successfully:

```bash
docker logs tsbridge --tail 20
```

You should see it registering `jellyfin.<tailnet>.ts.net` and `nextcloud.<tailnet>.ts.net` with no errors.

---

## Program Setup: Nextcloud AIO

From a browser on your LAN:

```
https://<SERVER_LAN_IP>:8080
```

Accept the certificate warning. When prompted for a domain, enter:

```
nextcloud.<tailnet>.ts.net
```

Complete setup and wait for all containers to start (~5 minutes). Nextcloud AIO will automatically join the `nextcloud-aio` Docker network and start the `nextcloud-aio-apache` container on port 11000, which tsbridge then proxies to your Tailnet.

If Nextcloud is unreachable via Tailscale after setup, check that tsbridge is on the `nextcloud-aio` network:

```bash
docker network inspect nextcloud-aio | grep tsbridge
```

If it's missing, run:

```bash
docker network connect nextcloud-aio tsbridge
```

Then make sure the `networks:` section is present in your `docker-compose.yml` so it persists across restarts.

---

## Program Setup: Samba

In File Explorer, right-click **Computer → Map Network Drive** and use:

```
\\<SERVER_LAN_IP>\myfiles
```

Enter the credentials you set in `docker-compose.yml` when prompted.

---

## Program Setup: Jellyfin

From a browser connected to your Tailnet:

```
https://jellyfin.<tailnet>.ts.net
```

Complete setup and map libraries to your media folders.

### Recommended file structure

**Movies** — one folder per film:

```
Movies
├── Fight Club
│   └── Fight Club [imdbid-tt0137523].mkv
└── Goodfellas
    └── Goodfellas [imdbid-tt0099685].mkv
```

**Shows** — series folder → season folder → episodes:

```
Shows
├── True Detective [imdbid-tt2356777]
│   └── Season 01
│       ├── True Detective S01E01.mkv
│       └── True Detective S01E02.mkv
└── Landman [imdbid-tt14186672]
    ├── Season 01
    │   ├── Landman S01E01.mkv
    │   └── Landman S01E02.mkv
    └── Season 02
        ├── Landman S02E01.mkv
        └── Landman S02E02.mkv
```

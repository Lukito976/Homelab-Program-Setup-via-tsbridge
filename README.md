# Homelab: Nextcloud AIO + Jellyfin + Samba via tsbridge (Tailscale)

This repository documents how to deploy a private homelab using Docker with:

* **[Nextcloud All-in-One (AIO)](https://github.com/nextcloud/all-in-one)** for cloud storage and collaboration
* **[Jellyfin](https://jellyfin.org/docs/general/installation/container/)** for media streaming
* **[Samba](https://github.com/dockur/samba)** for network file access (Windows/macOS/Linux)
* **[tsbridge](https://github.com/jtdowney/tsbridge)** to securely expose services over **Tailscale**
* **No public ports** and **no reverse proxy on the host**

All external access is restricted to devices on your Tailnet.

---

## Prerequisites

* Linux server (Tested on Ubuntu 24.04.3 LTS)
* Docker + Docker Compose installed
* [A Tailscale account](https://tailscale.com/)
* A Tailscale OAuth client with tag permissions

---

## Directory Layout

### Configuration files

```
~/homelab
├── docker-compose.yml
└── tsbridge.toml
```

### Shared media directory

```
/media/myfiles
├── Movies
└── Shows
```

---

## Step 1 — Install Docker

If [Docker](https://docs.docker.com/engine/install/ubuntu/) is not installed:

First, install the Docker apt repository:
```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

Then, install Docker:

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

## Step 2 — Create the Media Directory

All media for **Samba** & **Jellyfin** lives here:

```bash
sudo mkdir -p /media/myfiles
sudo chown -R 1000:1000 /media/myfiles
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

Do **not** rename it.
Do **not** let Docker Compose auto-prefix it.

---

## Step 5 — Configure Tailscale ACLs

Because tsbridge uses OAuth with tags, your Tailnet must allow the tag.

In **Tailscale Admin → Access Controls**, ensure:

```json
{
  "tagOwners": {
    "tag:tsbridge": ["YOUR_TAILSCALE_LOGIN_EMAIL"]
  }
}
```

Without this, tsbridge will fail to authenticate.

Next, generate a new a new OAuth Key:

Go to **Settings → Trust Credentials → + Credential (make sure to give it read and write privileges)**

Save the details of the key to populate the required files.

---

## Step 6 — Create `tsbridge.toml`

Create the config file:

```bash
nano ~/homelab/tsbridge.toml
```

```toml
[tailscale]
oauth_client_id = "YOUR_OAUTH_CLIENT_ID"
oauth_client_secret = "YOUR_OAUTH_CLIENT_SECRET"
state_dir = "/var/lib/tsbridge"
default_tags = ["tag:tsbridge"]

[[services]]
name = "jellyfin"
backend_addr = "http://jellyfin:8096"
tags = ["tag:tsbridge"]

[[services]]
name = "nextcloud"
backend_addr = "http://nextcloud-aio-apache:11000"
tags = ["tag:tsbridge"]
```

---

## Step 7 — Create `docker-compose.yml`

Create the file:

```bash
nano ~/homelab/docker-compose.yml
```

```yaml
services:
  tsbridge:
    image: ghcr.io/jtdowney/tsbridge:latest
    container_name: tsbridge
    command: ["-config", "/config/tsbridge.toml"]
    volumes:
      - tsbridge-state:/var/lib/tsbridge
      - /absolute/path/to/homelab/tsbridge.toml:/config/tsbridge.toml:ro
    restart: unless-stopped
    networks:
      - default
      - nextcloud-aio

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
      APACHE_PORT: 11000
      APACHE_IP_BINDING: 0.0.0.0
      SKIP_DOMAIN_VALIDATION: "true"

  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    volumes:
      - /media/myfiles:/media
      - jellyfin_config:/config
      - jellyfin_cache:/cache

  samba:
    image: dockurr/samba:latest
    container_name: samba
    restart: unless-stopped
    ports:
      - "445:445"
    volumes:
      - /media/myfiles:/storage
    environment:
      USER: "SAMBA_USERNAME"
      PASS: "SAMBA_PASSWORD"
      NAME: "myfiles"

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

## Step 8 — Start the Stack

```bash
cd ~/homelab
docker compose up -d
```

Verify ports:

```bash
sudo ss -tulpn | grep :8080
sudo ss -tulpn | grep :445
```

Everything should be working now.

---

## Program Setup: Nextcloud AIO

From a browser on the LAN:

  ```
  https://<SERVER_LAN_IP>:8080
  ```

* Accept the certificate warning
* When prompted for a domain, enter:

  ```
  nextcloud.<tailnet>.ts.net
  ```
  
* Complete setup and wait for all containers to start

---

## Program Setup: Samba

In File Explorer, right click on **Computer** and select **Map Network Drive** and populate it with the following: 

  ```
  \\<SERVER_LAN_IP>\myfiles
  ```

When prompted to enter login credentials, use the ones that you configured earlier in the `docker-compose.yml`.

Files written here appear immediately in Jellyfin; for ease of use, create a **Movies** and a **Shows** folder.

## Program Setup: Jellyfin

From a browser connected to your tailnet:

  ```
  https://jellyfin.<tailnet>.ts.net
  ```

* When prompted for the server name, enter your Ubuntu server name:

  ```
  Ubuntu-home-server-example
  ```
* Complete setup and map libraries to individual media folders (select the specific folder you created earlier in your Samba drive for each type of library).

* Movies should be organized into individual folders for each movie. See below for an example:
  
```
Movies
├── Fight Club 
│   └── Fight Club [imdbid-tt0137523].mkv
└── Goodfellas
    └── Goodfellas [imdbid-tt0099685].mkv 
```

* Shows should be organized into series folders, then into season folders under each series. See below for an example:
  
```
Shows
├── True Detective [imdbid-tt2356777]
│   ├── Season 01
│       ├── True Detective S01E01.mkv
│       └── True Detective S01E02.mkv
└── Landman [imdbid-tt14186672]
    ├── Season 01
    |   ├── Landman S01E01.mkv
    |   └── Landman S01E02.mkv
    └── Season 02
        ├── Landman S02E01.mkv
        └── Landman S02E02.mkv
```
---


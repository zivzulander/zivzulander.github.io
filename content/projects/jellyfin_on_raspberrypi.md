+++
title = "Jellyfin, Sonarr, Radarr and Deluge On Raspberry Pi 4"
author = ["Matt Morris"]
draft = false
+++

_Updated on September 01, 2022._

[Setup your pi]({{< relref "setup_pi" >}}) with 64-bit **Raspberry Pi OS** and mount your storage media.


## Install docker {#install-docker}

```bash
curl -sSL https://get.docker.com | sh
```

Add docker to the usergroup pi.

```bash
sudo usermod -aG docker pi
```

Make a folder for all the config data. I keep it all on my usb HDD.

```bash
mkdir /mnt/usb/docker
cd /mnt/usb/docker
```

Create and open the `docker-compose.yml` file.

```bash
nano docker-compose.yml
```

Copy the following, making adjustments as necessary for your file system. You'll
likely need to change the timezone and some of the volumes.

> NOTE: Docker volumes look like `/path/on/your/harddrive:/what/the/docker/container/sees` so change the part before the colon only.
>
> `./` means current directory which you can change to something like `/home/pi/radarr` if you want.

All containers are from [linuxserver.io](https://www.linuxserver.io/).

When finished press `Ctrl-S` to save and `Ctrl-X` to quit.

```Containerfile
---
version: "2.1"
services:
jellyfin:
    image: lscr.io/linuxserver/jellyfin
    container_name: jellyfin
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Edmonton
        #- JELLYFIN_PublishedServerUrl=192.168.0.5 #optional
    volumes:
      - ./jellyfin:/config
      - /mnt/usb/data:/data
      - ./garbage:/data/tvshows
      - ./garbage:/data/movies
      - /opt/vc/lib:/opt/vc/lib
      - /dev/shm:/config/data/transcoding-temp/transcodes
    ports:
      - 8096:8096
      - 8920:8920     #optional
      - 7359:7359/udp #optional
      - 1900:1900/udp #optional
    devices:
      #- /dev/vcsm-cma:/dev/vcsm-cma
      #- /dev/vchiq:/dev/vchiq
      - /dev/video10:/dev/video10
      - /dev/video11:/dev/video11
      - /dev/video12:/dev/video12
    restart: unless-stopped
  radarr:
    image: lscr.io/linuxserver/radarr
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Edmonton
    volumes:
      - ./radarr:/config
      - /mnt/usb/data/movies:/data/movies
      - /mnt/usb/data/downloads:/data/downloads #optional torrent downloads folder
    ports:
      - 7878:7878
    restart: unless-stopped
  sonarr:
    image: lscr.io/linuxserver/sonarr
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Edmonton
    volumes:
      - ./sonarr:/config
      - /mnt/usb/data/tv:/data/tv
      - /mnt/usb/data/downloads:/data/downloads #optional torrent downloads folder
    ports:
      - 8989:8989
    restart: unless-stopped
  deluge:
    image: lscr.io/linuxserver/deluge
    container_name: deluge
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Edmonton
    volumes:
      - ./deluge:/config
      - /mnt/usb/data/downloads:/data/downloads
    ports:
      - 8112:8112
      - 6881:6881
      - 6881:6881/udp
    restart: unless-stopped
  bazarr:
    image: lscr.io/linuxserver/bazarr:latest
    container_name: bazarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Edmonton
    volumes:
      - ./bazarr:/config
      - /mnt/usb/data/movies:/data/movies
      - /mnt/usb/data/tv:/data/tv
    ports:
      - 6767:6767
    restart: unless-stopped
```

Start all docker containers.

```bash
docker-compose up -d
```


## Configuring {#configuring}

You can now access all your web apps and begin to customize.

| <http://raspberryipaddress:7878> | jellyfin |
|----------------------------------|----------|
| <http://raspberryipaddress:7878> | radarr   |
| <http://raspberryipaddress:8989> | sonarr   |
| <http://raspberryipaddress:6767> | bazarr   |
| <http://raspberryipaddress:8112> | deluge   |

Your **raspberrypiIPADDRESS** can be found using ~hostname -I | grep -Eo '^&sect;\*'

Add your Movies and TV library. Since we forwarded the folders in docker, TV Shows are in `/data/tv` and movies in `/data/movies`.


### Radarr {#radarr}

Go to the web app <http://raspberryipaddress:7878> and click through the links:

Settings &gt; Media Management &gt; Add a Root Folder (at the very bottom)

Add `/data/movies`

Go to Settings &gt; Download Clients &gt; hit the big plus

Add deluge with the following settings:

| Host | raspberrypiIPADDRESS |
|------|----------------------|
| Port | 8112                 |


### Sonarr {#sonarr}

Go to the web app <http://raspberryipaddress:8989> and click through the links:

Settings &gt; Media Management &gt; Add a Root Folder (at the very bottom)

Add `/data/tv`

Go to Settings &gt; Download Clients &gt; hit the big plus

Add deluge with the following settings:

| Host | raspberrypiIPADDRESS |
|------|----------------------|
| Port | 8112                 |


### Deluge {#deluge}

Default user is **admin** and password **deluge**

Open Preferences and change the following:


#### Downloads {#downloads}

Change Download to: `/data/downloads`

Change Move completed to: `/downloads/completed` (optional)


#### Interface {#interface}

Change _webui_ password.

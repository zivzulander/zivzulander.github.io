+++
title = "Jellyfin, Sonarr, Radarr and Deluge On Raspberry Pi 4"
author = ["Matt Morris"]
draft = false
+++

## Install Jellyfin, Sonarr, Radarr and Deluge On a Raspberry Pi 4 {#install-jellyfin-sonarr-radarr-and-deluge-on-a-raspberry-pi-4}

Updated on August 16, 2022.
Read [here](https://gist.github.com/zivzulander/09c74ba8217d2a98dcbe1ca40653fc8a) for the basics of getting a Raspberry Pi setup for server use.


### Install docker and docker-compose {#install-docker-and-docker-compose}

Install `docker`.

```bash
curl -sSL https://get.docker.com | sh
```

Add docker to the usergroup pi.

```bash
sudo usermod -aG docker pi
```

Preferred way to install `docker-compose` (from [here](https://gist.github.com/blazewicz/04e666ae1f25387c8b291d81b12c550c)).

> NOTE: You can run these commands in one shot by pressing Ctrl-x-e (that's control `x` and keep holding control while pressing `e`) and pasting the lines as is.

```bash
sudo apt install python3-venv python3-dev libffi-dev libssl-dev
sudo mkdir -p /opt/local/docker-compose
sudo python3 -m venv /opt/local/docker-compose/venv
sudo /opt/local/docker-compose/venv/bin/pip install --upgrade pip
sudo /opt/local/docker-compose/venv/bin/pip install docker-compose
sudo ln -s /opt/local/docker-compose/venv/bin/docker-compose /usr/local/bin/docker-compose
```

Make a folder for all the config data. I just keep it all on my external HDD.

```bash
mkdir /mnt/usbHDD/docker
cd /mnt/usbHDD/docker
```

Create and open the `docker-compose.yml` file.

```bash
nano docker-compose.yml
```

Copy the following making adjustments as necessary for your file system. You'll likely need to change the timezone and some of the volumes.

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
      - /mnt/usbHDD/data:/media
      - ./garbage:/data/tvshows
      - ./garbage:/data/movies
      - /opt/vc/lib:/opt/vc/lib
      - /dev/shm:/config/data/transcoding-temp/transcodes
    ports:
      - 8096:8096
      - 8920:8920 #optional
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
      - ./radarr/config:/config
      - ./Movies:/movies
      - ./downloads:/downloads #optional torrent downloads folder
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
      - ./sonarr/config:/config
      - ./TV Shows:/tv
      - ./downloads:/downloads
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
      - /mnt/usbHDD/data/downloads:/data/downloads # where all your torrents will download
    ports:
      - 8112:8112
      - 6881:6881
      - 6881:6881/udp
    restart: unless-stopped
  tautulli:
    image: lscr.io/linuxserver/tautulli
    container_name: tautulli
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Edmonton
    volumes:
      - ./tautulli:/config
    ports:
      - 8181:8181
    restart: unless-stopped
```

Start all docker containers.

```bash
docker-compose up -d
```


### Configuring {#configuring}

You can now access all your web apps and begin to customize.

| <http://app.plex.tv>             | plex   |
|----------------------------------|--------|
| <http://raspberryipaddress:7878> | radarr |
| <http://raspberryipaddress:8989> | sonarr |
| <http://raspberryipaddress:8112> | deluge |

Your **raspberrypiIPADDRESS** can be found using `hostname -I | xargs -n 1`. It should be the first number unless you have multiple networks setup (e.g. ethernet and wifi).


#### Plex {#plex}

<http://raspberryipaddress:32400/web>

Add your Movies and TV library. Since we forwarded the folders in docker as `- ./TV Shows:/tv` this means that all our videos will be in `/path/to/docker-compose.yml/folder/TV Shows` which Plex sees as `/tv` so add `/tv` as the location of all your tv shows and `/movies` for all your movies.


#### Radarr {#radarr}

Go to the web app <http://raspberryipaddress:7878> and click through the links:

Settings &gt; Media Management &gt; Add a Root Folder (at the very bottom)

Add `/movies`

Go to Settings &gt; Download Clients &gt; hit the big plus

Add deluge with the following settings:

| Host | raspberrypiIPADDRESS |
|------|----------------------|
| Port | 8112                 |


#### Sonarr {#sonarr}

Go to the web app <http://raspberryipaddress:8989> and click through the links:

Settings &gt; Media Management &gt; Add a Root Folder (at the very bottom)

Add `/tv`

Go to Settings &gt; Download Clients &gt; hit the big plus

Add deluge with the following settings:

| Host | raspberrypiIPADDRESS |
|------|----------------------|
| Port | 8112                 |


#### Deluge {#deluge}

Default user is **admin** and password **deluge**

Open Preferences and change the following:

<!--list-separator-->

-  Downloads

    Change Download to: `/downloads`

    Change Move completed to: `/downloads/completed` (optional)

<!--list-separator-->

-  Interface

    Change _webui_ password.

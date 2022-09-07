+++
title = "Jellyfin, Sonarr, Radarr, Bazarr and Deluge On Raspberry Pi 4"
author = ["Matt Morris"]
draft = false
+++

_Updated on September 01, 2022._

The following will get you started with [Jellyfin](https://jellyfin.org/), [Radarr](https://radarr.video/), [Sonarr](https://sonarr.tv/), [Bazarr](https://www.bazarr.media/) and
[Deluge](https://www.deluge-torrent.org/) running on a Raspberry Pi. [Setup your pi]({{< relref "setup_pi" >}}) with 64-bit **Raspberry Pi OS** and
mount your storage media before continuing.


### Install docker {#install-docker}

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

Create and open a `docker-compose.yml` file.

```bash
nano docker-compose.yml
```

Copy the following, making adjustments as necessary for your file system. You'll
likely need to change the timezones and volumes.

> NOTE: Docker volumes look like `/path/on/your/harddrive:/what/the/docker/container/sees` so change the part before the colon only.
>
> `./` means current directory which you can change to something like `/home/pi/radarr` if you want.

```bash
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

When finished press `Ctrl-S` to save and `Ctrl-X` to quit.

Start all docker containers.

```bash
docker-compose up -d
```


### Configuring {#configuring}

You can now access all your web apps and begin to customize.

Your Raspberry Pi's ip address can be found using

```bash
hostname -I | grep -Eo '%\S*'
```


#### Jellyfin {#jellyfin}

<http://ipaddress:7878>

Add your Movies and TV library. Since we forwarded the folders in docker, TV Shows are in `/data/tv` and movies in `/data/movies`.


#### Radarr {#radarr}

Go to the web app <http://ipaddress:7878> and click through the links:

**Settings &gt; Media Management &gt; Add a Root Folder** (at the very bottom)

Add `/data/movies`

Go to **Settings &gt; Download Clients &gt;** hit the big plus

Add deluge with the following settings:

|      |                      |
|------|----------------------|
| Host | raspberrypiIPADDRESS |
| Port | 8112                 |


#### Sonarr {#sonarr}

Go to the web app <http://ipaddress:8989> and do the same as Radarr.


#### Deluge {#deluge}

<http://ipaddress:8112>

Default user is **admin** and password **deluge**

Open Preferences and change the following:

**Downloads**

Change Download to: `/data/downloads`

Change Move completed to: `/downloads/completed` (optional)

**Interface**

Change _webui_ password.


## Extras {#extras}


### Notifications on new downloads {#notifications-on-new-downloads}

You can get notifications through Jellyfin or through Radarr and Sonarr. I prefer the later. I use
[ntfy](https://ntfy.sh/) to get notifications on my phone.

Make a file called `jellyfin_notifications` and have it execute in Sonarr and Radarr

```bash
nano /mnt/usb/data/jellyfin_notifications
```

Copy in the following.

```bash
#!/bin/bash
ntfykey="your-random-key"
ip="http://ip-address-of-your-pi"
api="api-key-of-jellyfin" # generate one from dashboard > api

curl \
    -H "Title: $radarr_movie_title (${radarr_movie_year}) added" \
    ntfy.sh/$ntrykey > /dev/null 2>&1

case $sonarr_episodefile_episodecount in
    1) curl \
        -H "Title: New episode of $sonarr_series_title added" \
        -d "S${sonarr_episodefile_seasonnumber}xE$sonarr_episodefile_episodenumbers $sonarr_episodefile_episodetitles" \
        ntfy.sh/$ntfykey > /dev/null 2>&1
       ;;
    *) curl \
        -H "Title: $sonarr_series_title" \
        -d "Season $sonarr_episodefile_seasonnumber added" \
        ntfy.sh/$ntfykey > /dev/null 2>&1
       ;;
esac

# force jellyfin to scan for new media
curl -d "" http://$ip:8096/library/refresh?api_key=$api
```

Make it executable

```bash
sudo chmod +x /mnt/usb/data/jellyfin_notifications
```

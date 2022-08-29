+++
title = "Install Seafile on a Raspberry Pi"
author = ["Matt Morris"]
draft = false
+++

<style>.org-center { margin-left: auto; margin-right: auto; text-align: center; }</style>

<div class="org-center">

Updated on August 27, 2022 for [Seafile Server 9.0.2](https://github.com/haiwen/seafile-rpi) on Raspberry Pi Bullseye.

</div>

[Seafile](https://www.seafile.com/en/home/) is a free and open source file syncing tool that can be self-hosted on a raspberry pi. Below is a brief comparison of similar programs that are also free and self-hostable on a pi:

| Program   | Share files with others | Apps             | Speed |
|-----------|-------------------------|------------------|-------|
| Seafile   | yes                     | anything         | fast  |
| Nextcloud | yes                     | anything         | slow  |
| Syncthing | no                      | poor iOS support | fast  |

We'll be installing **Seafile** on a raspberry pi 4 and setting up a reverse proxy so we can connect to it remotely though our own domain. There are instructions on [Seafile's website](https://manual.seafile.com/deploy/) but the script and docker image don't seem to work on with the pi build. We'll be doing it all manually.


## Getting Started {#getting-started}

Click [here]({{< relref "setup_pi" >}}) to setup your pi with 64-bit **Raspberry Pi OS**.


### Getting started with your own domain {#getting-started-with-your-own-domain}

For security we're only going to access Seafile though https. You'll need your own domain. I use [namesilo](https://www.namesilo.com/) but there are many options. A domain such as yourdomain.xyz is about $1/year. You can even use the same address's subdomains for other applications such as bitwarden.yourdomain.xyz.

Once you have this you need to point dns records to your public ip. How this is done is a little different for each site but there should be an option to manage dns where you can add an `A record` pointing to your [public IPv4 address](https://whatismyipaddress.com/) and a `AAAA record` pointing to your IPv6 address. Something like this:

| Type | Domain | Data                     | TTL    |
|------|--------|--------------------------|--------|
| A    |        | 12.34.56.78              | 1 hour |
| AAAA |        | 1234:5a78:b90:c123::d4e5 | 1 hour |

If you leave the _domain_ (sometimes called _hostname_) blank then this will point to _yourdomain.xyz_. If you add a domain like _seafile_ then your Seafile will be on _seafile.yourdomain.xyz_. Which you choose is up to you.

| Type | Domain  | Data                     | TTL    |
|------|---------|--------------------------|--------|
| A    | seafile | 12.34.56.78              | 1 hour |
| AAAA | seafile | 1234:5a78:b90:c123::d4e5 | 1 hour |

It can take a few minutes to an hour for your dns server to refresh which is why we're doing this step first. You can run `ping -c1 seafile.yourdomain.xyz` to see if the IP matches your home ip address.


### Install MySQL {#install-mysql}

Update your pi if you didn't already..

```bash
sudo apt update && sudo apt upgrade -y
```

Install dependencies for `MySQL`.

```bash
sudo apt-get install -y python3 python3-setuptools python3-pip default-libmysqlclient-dev python3-pymysql memcached libmemcached-dev libffi-dev

sudo pip3 install --timeout=3600 django==3.2.* Pillow pylibmc captcha jinja2 sqlalchemy==1.4.3 \
    django-pylibmc django-simple-captcha python3-ldap mysqlclient pycryptodome==3.12.0 cffi==1.14.0 lxml pymysql
```

Set a MySQL root password which you'll be asked for later. You can just press enter on most other questions to accept the default response.

```bash
sudo mysql_secure_installation
```


### Download Seafile {#download-seafile}

Make a directory for Seafile to be installed to.

```bash
mkdir /mnt/usb/seafile
```

Download the Seafile script.

```bash
wget -qO- "https://github.com/haiwen/seafile-rpi/releases/download/v9.0.2/seafile-server-9.0.2-bullseye-arm64v8l.tar.gz" | tar xvz -C /mnt/usb/seafile
```


### Install Seafile {#install-seafile}

Run the installation script.

```bash
cd seafile-server-9.0.2

./setup-seafile-mysql.sh
```

Choose `[1] Create new ccnet/seafile/seahub databases`. Most of the questions have a default answer which is a good idea to use. Just hit enter on those. Use the root password from when you typed `sudo mysql_secure_installation`.


### Forward Ports on your router {#forward-ports-on-your-router}

We're using a reverse proxy so we only forward port 443 on your router to your raspberry pi's local ip.


### Configuring Seafile {#configuring-seafile}

Some config files need to be changed since we'll be accessing Seafile from our custom domain with https.


#### ccnet.conf {#ccnet-dot-conf}

```bash
nano /opt/seafile/conf/ccnet.conf
```

Change `SERVICE_URL` to your domain. e.g.

```bash
SERVICE_URL = https://seafile.example.com
```


#### seahub_settings.py {#seahub-settings-dot-py}

```bash
nano /opt/seafile/conf/seahub_settings.py
```

Change/add `FILE_SERVER_ROOT` to your domain and change your timezone.

```bash
FILE_SERVER_ROOT = 'https://seafile.example.com/seafhttp'
TIMEZONE = 'America/Denver'
```


#### seafdav.conf {#seafdav-dot-conf}

If you want webdav (you probably do) then change `enabled` to `true` and change the location of the folder with `share_name = /seafdav`. This will give you a working webdav server which is more widely available for syncing with apps. You can access this through <https://seafile.example.com/seafdav>

```bash
nano /opt/seafile/conf/seafdav.conf
```


### If you don't plan on using https {#if-you-don-t-plan-on-using-https}

This is a much less secure method and you will lose out on a lot of the benefits of Seafile such as sending links to others and accessing it remotely without VPNs but here it is.


### Reverse Proxy {#reverse-proxy}

This forwards http requests from <https://yourdomain.xyz> on port 443 to wherever Seafile is running. In our case, <http://127.0.0.1:8000>.

We'll go back the `pi` user to run the next steps.

```bash
exit
```


#### HTTPS certificate from [letsencrypt](https://letsencrypt.org/) {#https-certificate-from-letsencrypt}

Change YOURDOMAIN. It will place the files in `/etc/letsencrypt/live/YOURDOMAIN.XYZ/`

```bash
sudo apt install python3-certbot-nginx/stable -y
sudo certbot certonly --nginx --domain YOURDOMAIN.XYZ
```


#### Install `nginx` {#install-nginx}

```bash
sudo apt install nginx
sudo rm /etc/nginx/sites-{enabled,available}/default
touch /etc/nginx/sites-available/seafile.conf
sudo ln -s /etc/nginx/sites-available/seafile.conf /etc/nginx/sites-enabled/seafile.conf
```


#### Configuring `nginx` {#configuring-nginx}

This is the trickiest part if it's new to you. Copy the example below which is pieced together from [Seafile's website](https://manual.seafile.com/deploy/https_with_nginx/#modifying-nginx-configuration-file) and adjust appropriately. At the very least, everything is ALL CAPS should be changed.

```bash
sudo -e /etc/nginx/sites-available/seafile.conf
```

Here are some things to watch out for:

-   Replace _seafile.example.com_ with your domain each time it comes up
-   Change `root = /opt/seafile/seafile-server-latest/seahub;` in `location /media` to where you'll store all your files. I have this on an external drive `root = /mnt/usbHDD/seafile;`
-   This example contains a webdav config under `location /seafdav` so you can remove those lines if you aren't using it.

```nginx

log_format seafileformat '$http_x_forwarded_for $remote_addr [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" $upstream_response_time';

server {
    listen       80;
    server_name  SEAFILE.EXAMPLE.COM;
    rewrite ^ https://$http_host$request_uri? permanent;    # Forced redirect from HTTP to HTTPS

    server_tokens off;      # Prevents the Nginx version from being displayed in the HTTP response header
}
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    ssl_certificate /etc/letsencrypt/live/SEAFILE.EXAMPLE.COM/fullchain.pem;    # Path to your fullchain.pem
    ssl_certificate_key /etc/letsencrypt/live/SEAFILE.EXAMPLE.COM/privkey.pem;  # Path to your privkey.pem
    server_name SEAFILE.EXAMPLE.COM;
    server_tokens off;

    location / {
         proxy_pass         http://127.0.0.1:8000;
         proxy_set_header   Host $host;
         proxy_set_header   X-Real-IP $remote_addr;
         proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
         proxy_set_header   X-Forwarded-Host $server_name;
         proxy_read_timeout  1200s;

         # used for view/edit office file via Office Online Server
         client_max_body_size 0;

         access_log      /var/log/nginx/seahub.access.log seafileformat;
         error_log       /var/log/nginx/seahub.error.log;
    }

    location /seafhttp {
        rewrite ^/seafhttp(.*)$ $1 break;
        proxy_pass http://127.0.0.1:8082;
        client_max_body_size 0;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_connect_timeout  36000s;
        proxy_read_timeout  36000s;
        proxy_send_timeout  36000s;
        proxy_request_buffering off; # enables large file uploads

        send_timeout  36000s;

        access_log      /var/log/nginx/seafhttp.access.log seafileformat;
        error_log       /var/log/nginx/seafhttp.error.log;
    }
     location /seafdav {

        client_max_body_size 0;
        proxy_connect_timeout  36000s;
        proxy_read_timeout  36000s;
        proxy_send_timeout  36000s;
        send_timeout  36000s;

        proxy_request_buffering off;

        access_log      /var/log/nginx/seafdav.access.log;
        error_log       /var/log/nginx/seafdav.error.log;
    }
    location /media {
        root /mnt/usb/seafile/seafile-server-latest/seahub;
    }
}

```

Start `nginx.`

```bash
sudo systemctl enable --now nginx
```


### Starting Seafile {#starting-seafile}

We want Seafile to start on boot. Create these 2 systemd unit files.

```bash
sudo -e /etc/systemd/system/seafile.service
```

Paste in the following:

```bash
[Unit]
Description=Seafile
After=network.target mysql.service

[Service]
Type=forking
ExecStart=/opt/seafile/seafile-server-latest/seafile.sh start
ExecStop=/opt/seafile/seafile-server-latest/seafile.sh stop
LimitNOFILE=infinity
User=seafile
Group=seafile

[Install]
WantedBy=multi-user.target
```

And

```bash
sudo -e /etc/systemd/system/seahub.service
```

```bash
[Unit]
Description=Seafile hub
After=network.target seafile.service

[Service]
Environment="LC_ALL=C"
Type=forking
ExecStart=/opt/seafile/seafile-server-latest/seahub.sh start
ExecStop=/opt/seafile/seafile-server-latest/seahub.sh stop
User=seafile
Group=seafile

[Install]
WantedBy=multi-user.target
```


#### Start and enable Seafile {#start-and-enable-seafile}

have to run seahub first

```bash
sudo systemctl enable --now seafile seahub
```

You should now be able to access Seafile from the https domain. Login with the username and password you provided to the `./setup-seafile-mysql.sh` script.


### Extra Stuff {#extra-stuff}


#### Enable Two Factor Authentication {#enable-two-factor-authentication}

**NOTE: Does not work with WebDAV**

Click on your avatar and got to _System Admin &gt; Settings_.

Check the box titled `Enable two factor authentication`.


#### Add more users {#add-more-users}

Click on your avatar and got to _System Admin &gt; Users_.


#### Enable Fail2Ban {#enable-fail2ban}

[Read more here.](https://manual.seafile.com/security/fail2ban/)

Make sure you added your timezone when changing [seahub_settings.py](#seahub-settings-dot-py)

Install it and edit the `jail.local` file.

```bash
sudo apt install fail2ban
sudo cp /etc/fail2ban/jail.{conf,local}
sudo -e /etc/fail2ban/jail.local
```

Add the following at the bottom changing the logpath if needed.

```bash
[seafile]
enabled  = true
port     = http,https
filter   = seafile-auth
logpath  = /opt/seafile/logs/seahub.log
maxretry = 3
```

```bash
sudo -e /etc/fail2ban/filter.d/seafile-auth.conf
```

Add the following:

```bash
# Fail2Ban filter for seafile
#

[INCLUDES]

# Read common prefixes. If any customizations available -- read them from
# common.local
before = common.conf

[Definition]

_daemon = seaf-server

failregex = Login attempt limit reached.*, ip: <HOST>

ignoreregex =

# DEV Notes:
#
# pattern :     2015-10-20 15:20:32,402 [WARNING] seahub.auth.views:155 login Login attempt limit reached, username: <user>, ip: 1.2.3.4, attemps: 3
#       2015-10-20 17:04:32,235 [WARNING] seahub.auth.views:163 login Login attempt limit reached, ip: 1.2.3.4, attempts: 3

```

Restart `fail2ban`

```bash
sudo fail2ban-client reload
```


#### Automated Garbage Collection {#automated-garbage-collection}

i.e. emptying the trash

We'll have to write a little script for this since we want to automate it and it requires a few steps.

```bash
sudo -e /home/pi/.local/bin/cleanup_seafile
```

Paste in the following. Change the path to Seafile if necessary.

```bash
#!/bin/bash

# stop the server
systemctl stop seafile seahub

sleep 20

sudo -u seafile /opt/seafile/seafile-server-latest/seaf-gc.sh

sleep 60

# start the server
systemctl start seafile seahub

```

Make this file executable:

```bash
sudo chmod +x /home/pi/.local/bin/cleanup_seafile
```

Again, we'll use `crontab` (still in user `pi`) to automate running this script.

```bash
crontab -e
```

Run at 4am first of each month.

```bash
0 4 * */1 *  sudo /home/pi/.local/bin/cleanup_seafile
```

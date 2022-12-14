+++
title = "Install Seafile on a Raspberry Pi"
author = ["Matt Morris"]
draft = false
+++

<style>.org-center { margin-left: auto; margin-right: auto; text-align: center; }</style>

<div class="org-center">

_Updated on September 07, 2022 for [Seafile Server 9.0.2](https://github.com/haiwen/seafile-rpi) on Raspberry Pi Bullseye._

</div>

[Seafile](https://www.seafile.com/en/home/) is a free and open source file syncing tool that can be self-hosted on a raspberry pi. While there are other tools you can use to sync your data, Seafile has a few advantages:

-   access data remotely over https
-   send or share files and folders with other users or guests
-   edit documents right in the browser
-   simple WebDAV setup
-   2FA
-   iOS clients
-   data de-duplication
-   multi-user support

There is one well-known disadvantage: Files are stored in a proprietary format
which can't be accessed through a normal file browser. You have to either mount
it or run a [script](https://manual.seafile.com/maintain/seafile_fsck/) to convert the data.

You could hack together your own server with [Syncthing](https://syncthing.net/), [rclone mount](https://rclone.org/commands/rclone_mount/), [nginx
WebDAV](https://nginx.org/en/docs/http/ngx_http_dav_module.html), etc., but it will probably be a lot more trouble.

We'll be installing **Seafile** on a Raspberry Pi 4 and setting up a reverse proxy
so we can connect to it remotely though our own domain. There are instructions
on [Seafile's website](https://manual.seafile.com/deploy/) but the script and docker image don't seem to work on with
the pi build at this time. We'll be installing it manually.

> There is a script I wrote to automate the whole process, available on my Github. It hasn't been well tested so use at own risk.
>
> <https://github.com/zivzulander/scripts/blob/master/seafile_install>


## Getting started {#getting-started}

[Setup your pi]({{< relref "setup_pi" >}}) with 64-bit **Raspberry Pi OS**.


## Setting up your own domain {#setting-up-your-own-domain}

For security we're only going to access seafile though https. You'll need your
own domain. I use [Namesilo](https://www.namesilo.com/) but there are many options. A domain such as
<https://yourdomain.xyz> is about $1/year. You can even use the same address's subdomains
for other applications such as bitwarden.yourdomain.xyz.

Once you have this you need to point dns records to your public ip. How this is
done is a little different for each site but there should be an option to manage
dns where you can add an `a record` pointing to your [public ipv4 address](https://whatismyipaddress.com/) and a
`aaaa record` pointing to your ipv6 address. Something like this:

| type | domain | data                     | ttl    |
|------|--------|--------------------------|--------|
| a    |        | 12.34.56.78              | 1 hour |
| aaaa |        | 1234:5a78:b90:c123::d4e5 | 1 hour |

If you leave the _domain_ (sometimes called _hostname_) blank then this will point
to _YOURDOMAIN.XYZ_. If you add a domain like _seafile_ then your seafile will be on
_seafile.YOURDOMAIN.XYZ_. Whichever you choose is up to you.

| type | domain  | data                     | ttl    |
|------|---------|--------------------------|--------|
| a    | seafile | 12.34.56.78              | 1 hour |
| aaaa | seafile | 1234:5a78:b90:c123::d4e5 | 1 hour |

It can take a few minutes to an hour for your dns server to refresh which is why
we're doing this step first. You can run `ping -c1 seafile.YOURDOMAIN.XYZ` to see
if the ip matches your home ip address.


## Installing prerequisites {#installing-prerequisites}

Update your pi if you didn't already.

```bash
sudo apt update && sudo apt upgrade -y
```

Since we are going to be exposing this server the internet, we should make sure
security updates are installed automatically.

```bash
sudo apt install unattended-upgrades -y
```

Install dependencies for `MySQL` as well as a few other tools we'll be using.

```bash
sudo apt install -y python3 python3-setuptools python3-pip default-libmysqlclient-dev python3-pymysql memcached libmemcached-dev libffi-dev python3-certbot-nginx/stable nginx fail2ban
```

Install other python prerequisites.

```bash
sudo pip3 install --timeout=3600 django==3.2.* pillow pylibmc captcha jinja2 sqlalchemy==1.4.3 \
    django-pylibmc django-simple-captcha python3-ldap mysqlclient pycryptodome==3.12.0 cffi==1.14.0 lxml pymysql
```

Set a MySQL root password which you'll be asked for later. You can just press
enter on the other questions to accept the default response.

```bash
sudo mysql_secure_installation
```


## Download seafile {#download-seafile}

Make a directory where Seafile and all your data will installed. I install it on a usb HDD:

```bash
mkdir /mnt/usb/seafile
```

Download the latest Seafile scripts.

```bash
wget -qo- "https://github.com/haiwen/seafile-rpi/releases/download/v9.0.2/seafile-server-9.0.2-bullseye-arm64v8l.tar.gz" | tar xvz -c /mnt/usb/seafile
```


## Install Seafile {#install-seafile}

Run the installation script.
_would this put it in pwd?_

```bash
/mnt/usb/seafile/setup-seafile-mysql.sh
```

Choose `[1] create new ccnet/seafile/seahub databases`. Most of the questions have
a default answer which is a good idea to use. Just hit enter on those. Use the
root password from when you typed `sudo mysql_secure_installation`.


## Forward ports on your router {#forward-ports-on-your-router}

We're using a reverse proxy with `certbot` so we forward port 80 and 443 on your
router to your Raspberry Pi's local ip.


## Configuring Seafile {#configuring-seafile}

Some config files need to be changed since we'll be accessing Seafile from our
custom domain through https.


### ccnet.conf {#ccnet-dot-conf}

```bash
$EDITOR /opt/seafile/conf/ccnet.conf
```

Change `service_url` to your domain. e.g.

```bash
service_url = https://YOURDOMAIN.XYZ
```


### seahub_settings.py {#seahub-settings-dot-py}

```bash
$EDITOR /opt/seafile/conf/seahub_settings.py
```

Change/add `file_server_root` to your domain and change your timezone (you can look for yours [here](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)).

```bash
file_server_root = 'https://YOURDOMAIN.XYZ/seafhttp'
timezone = 'America/Denver'
```


### seafdav.conf {#seafdav-dot-conf}

If you want WebDAV (note that you cannot use with 2FA) then change `enabled` to
`true` and change the location of the folder with `share_name = /seafdav`. This will
give you a working WebDAV server which is more widely available for accessing
your data. You can access this through <https://yourdomain.xyz/seafdav>

```bash
$EDITOR /opt/seafile/conf/seafdav.conf
```


## Reverse Proxy {#reverse-proxy}

This forwards requests from <https://yourdomain.xyz> to wherever Seafile is running.


### HTTPS certificate from [letsencrypt](https://letsencrypt.org/) {#https-certificate-from-letsencrypt}

Change YOURDOMAIN.XYZ to your domain. It will place the certificates in
`/etc/letsencrypt/live/YOURDOMAIN.XYZ/`. Remember that port 80 must be open for
validation and that no other web server can be running.

```bash
sudo certbot certonly --nginx --domain YOURDOMAIN.XYZ
```


### Configuring `nginx` {#configuring-nginx}

First things first.

```bash
sudo rm /etc/nginx/sites-{enabled,available}/default

touch /etc/nginx/sites-available/seafile.conf

sudo ln -s /etc/nginx/sites-available/seafile.conf /etc/nginx/sites-enabled/seafile.conf

sudo -e /etc/nginx/sites-available/seafile.conf
```

This can be the trickiest part if it's new to you. Copy the below example which
is pieced together from [Seafile's website](https://manual.seafile.com/deploy/https_with_nginx/#modifying-nginx-configuration-file) and adjust appropriately.

Here are some things to watch out for:

-   Replace _YOURDOMAIN.XYZ_ with your domain
-   Change `root = /mnt/usb/seafile/seafile-server-latest/seahub;` in `location /media` to where you'll store all your files.
-   This example contains a WebDAV config under `location /seafdav` so you can remove those lines if you aren't using it.

```nginx

log_format seafileformat '$http_x_forwarded_for $remote_addr [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" $upstream_response_time';

server {
    listen       80;
    server_name  YOURDOMAIN.XYZ;
    rewrite ^ https://$http_host$request_uri? permanent;    # Forced redirect from HTTP to HTTPS

    server_tokens off;      # Prevents the Nginx version from being displayed in the HTTP response header
}
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    ssl_certificate /etc/letsencrypt/live/YOURDOMAIN.XYZ/fullchain.pem;    # Path to your fullchain.pem
    ssl_certificate_key /etc/letsencrypt/live/YOURDOMAIN.XYZ/privkey.pem;  # Path to your privkey.pem
    server_name YOURDOMAIN.XYZ;
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

Test that the configuration works

```bash
sudo nginx -t
```

Start (or restart) `nginx.`

```bash
sudo systemctl enable --now nginx

or

sudo systemctl restart nginx
```


## Starting Seafile {#starting-seafile}

We want Seafile to start on boot. Create these 2 systemd unit files.

```bash
sudo -e /etc/systemd/system/seafile.service
```

Paste in the following (change directories if necessary):

**change user**

```bash
[Unit]
Description=Seafile
After=network.target mysql.service

[Service]
Type=forking
ExecStart=/mnt/usb/seafile/seafile-server-latest/seafile.sh start
ExecStop=/mnt/usb/seafile/seafile-server-latest/seafile.sh stop
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
ExecStart=/mnt/usb/seafile/seafile-server-latest/seahub.sh start
ExecStop=/mnt/usb/seafile/seafile-server-latest/seahub.sh stop
User=seafile
Group=seafile

[Install]
WantedBy=multi-user.target
```


### Start and enable Seafile {#start-and-enable-seafile}

_have to run seahub first?_

```bash
sudo systemctl enable --now seafile seahub
```

You should now be able to access Seafile from the https domain. Login with the
username and password you provided to the `./setup-seafile-mysql.sh` script.


## Extra Stuff {#extra-stuff}


### Enable Fail2Ban {#enable-fail2ban}

[Read more here.](https://manual.seafile.com/security/fail2ban/)

Make sure you added your timezone when changing [seahub_settings.py](#seahub-settings-dot-py)

Backup then edit the `jail.local` file.

```bash
sudo cp /etc/fail2ban/jail.{conf,local}
sudo -e /etc/fail2ban/jail.local
```

Add the following at the bottom (change the logpath if needed):

```bash
[seafile]
enabled  = true
port     = http,https
filter   = seafile-auth
logpath  = /mnt/usb/seafile/logs/seahub.log
maxretry = 3
```

Copy and paste the following right in the terminal.

```bash
cat << EOF | sudo tee /etc/fail2ban/filter.d/seafile-auth.conf
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
EOF
```

Restart `fail2ban`

```bash
sudo fail2ban-client reload
```


### Automated Garbage Collection {#automated-garbage-collection}

(Emptying the trash)

We'll have to write a little script for this since we want to automate it and it requires a few steps.

```bash
sudo -e /usr/local/bin/cleanup_seafile
```

Paste in the following. Change the path to Seafile if necessary.
**need seafile user stuff and sudo?**

```bash
cat << EOF | sudo tee /usr/local/bin/cleanup_seafile
#!/bin/bash

# stop the server
systemctl stop seafile seahub

sleep 20

sudo -u seafile /mnt/usb/seafile/seafile-server-latest/seaf-gc.sh

sleep 60

# start the server
systemctl start seafile seahub
EOF
```

Make this file executable:

```bash
sudo chmod +x /usr/local/bin/cleanup_seafile
```

We'll use `crontab` to automate running this script.

```bash
crontab -e
```

For example, to have it run at 4am on the first of each month. See [here](https://crontab.guru/) for other options.

```bash
0 4 1 * *  sudo /usr/local/bin/cleanup_seafile
```


### Enable Two Factor Authentication {#enable-two-factor-authentication}

**NOTE: Does not work with WebDAV**

Click on your avatar and got to _System Admin &gt; Settings_.

Check the box titled `Enable two factor authentication`.


### Add more users {#add-more-users}

Click on your avatar and got to _System Admin &gt; Users_.

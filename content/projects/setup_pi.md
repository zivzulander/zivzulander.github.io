+++
title = "Setup Your Raspberry Pi"
author = ["Matt Morris"]
tags = ["linux", "raspberrypi", "tutorial"]
draft = false
+++

## Installing the OS to an SD Card {#installing-the-os-to-an-sd-card}

Install the [Raspberry Pi Imager](https://www.raspberrypi.com/software/). Under _CHOOSE OS_ select _Raspberry Pi OS (Other)_ and _Raspberry Pi OS Lite (64-bit)_. Click the settings cog to setup wifi, enable ssh, change the default password and change locale settings. Under _CHOOSE STORAGE_ select your inserted sd card.

{{< figure src="/blog-images/rpi-imager.png" >}}


## Find your pi's ip address {#find-your-pi-s-ip-address}

[Raspberry Pi Documentation - Remote Access](https://www.raspberrypi.com/documentation/computers/remote-access.html#ip-address)
Boot up the pi and find your new ip address. There are a few ways to do this:

-   Using the [Adafruit-Pi-Finder](https://github.com/adafruit/Adafruit-Pi-Finder/).
-   using `nmap -sn 10.0.0.1/24 | grep -Po '(?<=Nmap scan report for ).*'`
    Replace `10.0.0.1/24` with your local network's range.
-   you can also try to ping your pi using its hostname `ping raspberrypi`


## First steps {#first-steps}

Once you have your pi booted up and ssh'd into it you'll want to update your pi (may take a while).

```bash
sudo apt update && sudo apt upgrade -y
```

Let's set up a `crontab` so this runs once a week.

```bash
crontab -e
```

Adding this line to the end of this file runs the command at 4:30am every Tuesday. If you want to change this time you can check some other crontab options [here](https://crontab.guru/).

```bash
30 4 * * 2 sudo apt update && sudo apt upgrade -y
30 3 19 * * sudo apt autoremove # removes unused packages to free up space on the 19th of each month
```


## Mount USB hard drive {#mount-usb-hard-drive}


### Mount your hard drive {#mount-your-hard-drive}

Find the device you want to mount.

```bash
lsblk -f
```

Which outputs something like:

```bash
NAME        FSTYPE FSVER LABEL  UUID                                 FSAVAIL FSUSE% MOUNTPOINT
sda
└─sda1      ext4   1.0          6f9a255b-d43f-4877-94e9-4c6aacfcce7b  559.1G    80%
mmcblk0
├─mmcblk0p1 vfat   FAT32 boot   C839-E506                               203M    19% /boot
└─mmcblk0p2 ext4   1.0   rootfs 568caafd-bab1-46cb-921b-cd257b61f505     10G    26% /
```

For me it's `sda1`  on the 2nd line so we use `/dev/sda1` (it will always be `/dev/sdX` followed by a number where X is a lowercase letter). Copy this UUID which we'll use in a sec.


### Format the drive (skip to [Mount the usb drive](#mount-the-usb-drive) if you don't need this) {#format-the-drive--skip-to-mount-the-usb-drive--org260b033--if-you-don-t-need-this}

It's always good to start with a fresh drive and it will be a lot easier if it's not formatted for a macos or Windows filesystem.

```bash
sudo fdisk /dev/sdX
```

This runs the `fdisk` program which is the easiest way to do this. Pressing `m` and hitting enter shows the basic commands. The important ones for us are:

| d | delete a partition           |
|---|------------------------------|
| n | add a new partition          |
| w | write table to disk and exit |

Press `d` to delete any partitions. If there's only 1 it will delete it, if there are more than one it will ask which one to delete. Delete the rest if there are any.

Press `n` to add a new partition. It will ask various questions like

```bash
Partition number (1-128, default 1):
```

Which means the default is **1** and if we just press enter we are accepting the default. Do this for each question. It may also say something about an ext4 or Windows signature that will be removed. You likely don't need this so say yes to remove it.

Press `w` to save and quit. This will obviously erase everything on this disk so make sure you didn't run `fdisk` on the wrong disk.

Use `lsblk -f` to see the formatted drive and get its UUID.


### Mount the usb drive {#mount-the-usb-drive}

Create a folder on your SD card to mount the drive. Here I'm using /mnt/usb.

```bash
sudo mkdir /mnt/usb && sudo mount /dev/sda1 /mnt/usb
```

Set the disk to mount after a reboot. Start by getting this new drive's UUID:

```bash
lsblk -f /dev/sdX
```

Open `fstab` in a text editor.

```bash
sudo -e /etc/fstab
```

When you're done editing press `Ctrl-S` to save and `Ctrl-X` to quit.

Add the following with your drive's UUID copied from before.

```bash
UUID=YOUR-UUID	/mnt/usb	ext4	defaults	0	0
```


## Secure SSH {#secure-ssh}

You probably don't want to have to type in your password each time plus ssh keys are more secure. Type the following on your host.

```bash
ssh-copy-id YOUR-PI'S-IP-ADDRESS
```

Then change the following line in /etc/ssh/sshd_config


## Installing manually {#installing-manually}

Download the image

```bash
wget gg
```


### Enable ssh {#enable-ssh}

Fine the `boot` drive and mount it.

```bash
lsblk
```

gives an output like

```bash
NAME          MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda             8:0    0  113G  0 disk
├─sda1          8:1    0  511M  0 part  /boot
├─sda2          8:2    0 19.5G  0 part  /
└─sda3          8:3    0   93G  0 part
  └─ainstsda3 254:0    0   93G  0 crypt /home
sdb             8:16   1 14.5G  0 disk
├─sdb1          8:17   1  256M  0 part
└─sdb2          8:18   1  1.6G  0 part  /mnt
zram0         253:0    0  3.9G  0 disk  [SWAP]
```

`sbd` is my SD card and `sdb2`, the larger partition is where the boot folder is.

```bash
sudo mount /dev/sdb2 /mnt
touch /mnt/boot/ssh
sudo umount /dev/sdb2
```

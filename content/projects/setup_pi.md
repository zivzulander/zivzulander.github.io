+++
title = "Setup Your Raspberry Pi"
author = ["Matt Morris"]
draft = true
+++

Date: <span class="timestamp-wrapper"><span class="timestamp">August 20, 2022</span></span>


## Installing the OS to an SD Card {#installing-the-os-to-an-sd-card}

Install the [Raspberry Pi Imager](https://www.raspberrypi.com/software/). Under _CHOOSE OS_ select _Raspberry Pi OS (Other)_ and _Raspberry Pi OS Lite (64-bit)_. Click the settings cog to setup wifi, enable ssh, change the default password and change locale settings. Under _CHOOSE STORAGE_ select your inserted sd card.

{{< figure src="/blog-images/rpi-imager.png" >}}


## First steps {#first-steps}

Once you have your pi booted up and ssh'd into it you'll want to update your pi (may take a while).

<style>.org-center { margin-left: auto; margin-right: auto; text-align: center; }</style>

<div class="org-center">

_If you aren't sure how to find your pi's ip address, check the_ [official documentation.](https://www.raspberrypi.com/documentation/computers/remote-access.html#ip-address)

</div>

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


### Format the drive (skip to [Mount the usb drive](#mount-the-usb-drive) if you don't need this) {#format-the-drive--skip-to-mount-the-usb-drive--org07bfaaf--if-you-don-t-need-this}

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
lsblk -f /dev/sda1
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

The following increases security but you weill no longer be able to login with just a password so try logging out and back in without a password to ensure it works.

The following lines should be uncommented (remove the #) and changed accordingly in `/etc/ssh/sshd_config`

```bash
PubkeyAuthentication yes # it's actually available by default

PasswordAuthentication no
```

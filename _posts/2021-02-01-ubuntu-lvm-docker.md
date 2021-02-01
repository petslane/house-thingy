---
layout: post
title:  "Ubuntu, LVM and docker setup on Raspberry Pi 4"
excerpt_separator: <!--more-->
---

It took some time but I finally received my Raspberry pi 4 8GB.
Armor case for it arrived much faster. This Armor case is without any fans.
I have used successfully the same passive cooling case for the 1GB version of Pi4.
Within the last 6 months, 1GB Pi4 CPU temperature has been hovering between 50...55C.

<!--more-->

## Preparing Ubuntu

Time to setup OS. On my servers I have always used Ubuntu, currently, 1GB Pi4 uses Ubuntu,
so again, I chose [Ubuntu 20.04.1 server LTS 64bit](https://ubuntu.com/download/raspberry-pi).

I downloaded Ubuntu image and wrote it to SD card and booted up Pi4. On first boot Ubuntu updates itself, so for me, it took over half an hour, while it updated itself I could not install any new packages. After it finished I removed Snap based on [this](https://www.kevin-custer.com/blog/disabling-snaps-in-ubuntu-20-04/) instructions, just because I do not like it. I did not have to do `umount` part, probably because it was a fresh install of Ubuntu.

Then I installed Raspberry Pi specific software and updated firmware:
```
$ sudo apt install rpi-eeprom libraspberrypi-bin
$ sudo rpi-eeprom-update -a
BCM2711 detected
VL805 firmware in bootloader EEPROM
*** INSTALLING EEPROM UPDATES ***
BOOTLOADER: update available
CURRENT: Thu Mar 19 14:27:25 UTC 2020 (1584628045)
 LATEST: Thu Sep  3 12:11:43 UTC 2020 (1599135103)
 FW DIR: /lib/firmware/raspberrypi/bootloader/default
VL805: up-to-date
CURRENT: 000137ad
 LATEST: 000137ad
BOOTFS /boot/firmware
EEPROM updates pending. Please reboot to apply the update.
$ sudo reboot
```

## SATA to USB3 adapter

Now it's time to add SSD. First, we need to attach SSD with a SATA-to-USB3 adapter. I bought mine for a couple of euros from china. But these cheap adapters do not work properly without enabling quirks mode. More about this [here](https://www.raspberrypi.org/forums/viewtopic.php?t=245931). To enable quirks mode, you have to edit `/boot/firmware/cmdline.txt` file, but this change will enable quirks mode only on the next boot. To enable quirks mode now without a reboot, you have to write the same value you put for `usb-storage.quirks` in `cmdline.txt` file into `/sys/module/usb_storage/parameters/quirks` file. For example, if `lsusb` identifies your SATA adapter as `2109:3431`, then add `usb-storage.quirks=2109:3431:u` to `cmdline.txt` file and write `2109:3431:u` into `/sys/module/usb_storage/parameters/quirks` file. If you have multiple adapters with different VID/PID, then write them all into both files separated with comma, eg `152d:0578:u,2109:0711:u,1f75:0621:u`.

## Moving to SSD

If needed, remove all existing partitions from dist using `fdisk`.

To start using LVM, we first need to create physical volume (PV) with command `pvcreate /dev/sda`. Then on top of that, we need volume group (VG) that we name as `pivg`, using command `vgcreate pivg /dev/sda`.

Now let's create partitions, or logical volumes (LV). We start with one LV for root filesystem. Using command `lvcreate -L 10G pivg -n piroot` we create LV named `piroot` into `pivg` VG with 10GB in size. To use that partition, we need some filesystem. I considered trying out `btrfs` because it supports online shrinking, something that `ext4` does not support - that is while filesystem is in use, it's possible to reduce the size of filesystem. Both filesystems support online growing. But because integration between `btrfs` and `lvm` is not very good, then I decided to stick with `ext4` for now and maybe try out `btrfs` next time. `btrfs` and `lvm` work together, but if I what to increase LV and FS size, then I can do it with one command for `ext4`, but with `btrfs` I have to separately increase LV and FS size. To create `ext4` filesystem, use command `mkfs.ext4 /dev/mapper/pivg-piroot`. To copy current root files to new fs, create `/media/root` folder, mount it with `mount /dev/mapper/pivg-piroot /media/root` and copy all root fs files into mounted folder with command `rsync -axHAWXS --numeric-ids --info=progress2 / /media/root/`. Unmount `/media/root`.

Now there are 2 root filesystems, one on SD card and another on SSD. `cmdline.txt` has config that specifies what FS to use as system root. By default it's `root=LABEL=writable`, this needs to be changed to `root=/dev/mapper/pivg-piroot` and `rootdelay=5` must be added. In `fstab` file, replace `LABEL=writable...` line with `/dev/mapper/pivg-piroot / ext4 defaults,noatime 0 0`. And reboot.

After new startup you can verify that root fs on SSD is used with command `lsblk`. You can now delete old root fs from SD card with `fdisk`. Boot fs remains on SD card, this is OK as it is used on startup only and with no writing on that partition, SD card should not fail, but having a backup of this SD card would be a good idea. How to backup/restore SD card and how to setup LVM raid 1 will be shown in a future post.

## More partitions

Now it's time to create additional partitions. Although my Pi4 has 8GB of ram, I would like to add an additional swap in case there is a need for additional memory at some point. For that, we need to create a new LV with LVM and format it as swap fs. Using command `lvcreate -L 8G pivg -n piswap`, `mkswap /dev/mapper/pivg-piswap` and `swapon /dev/mapper/pivg-piswap` we have created LV, formatted it into swap fs and mounted it, so system is now using it. To mount it automatically after reboots, we have to edit `/etc/fstab` file and add `/dev/mapper/pivg-piswap none swap sw 0 0` row.

I am planning to use docker and I do not what to compromise system stability by accidentally running out of space in root filesystem because docker images can take a lot of space if you do not keep an eye on it. To avoid that, we can create a separate partition for docker and we can add additional space to it whenever it's required. Similarly to swap and root LV, we will create LV for docker - `lvcreate -L 10G pivg -n pidocker`, `mkfs.ext4 /dev/mapper/pivg-pidocker`. Mount it under `/mnt/docker` and add `/dev/mapper/pivg-pidocker /mnt/docker ext4 defaults,noatime 0 1` into `fstab` file. If we need to increase docker filesystem, then we can do it with command `lvresize --resizefs --size +5GB /dev/mapper/pivg-pidocker`, this increases filesystem by 5GB. With the same command we can shrink our fs, just specify `--size` argument with a negative value to shrink, eg `-5GB` to shrink by 5GB, or just `7GB` for desired final size. But as said before, shrinking ext4 fs can be done only if it's unmounted, this means we have to power off Raspberry and attached SSD to some other Linux system and do that command there. We can now create `/etc/docker/daemon.json` file and add `{"data-root":"/mnt/docker"}`, so after we install docker, it will immediately start using the new docker folder.

## Installing Docker

Installing docker is quite easy. To install Docker, just run `curl -sSL https://get.docker.com | sh` and that's it.

I use `docker-compose` to manage Docker containers. To install it, we first need to install some dependencies - `apt-get install -y libffi-dev libssl-dev python3 python3-pip` and then install `docker-compose` itself with command `pip3 -v install docker-compose`.

By default, in Raspberry Pi version of Ubuntu, when you do `docker stats`, then it does not show memory usage. Add `cgroup_enable=memory cgroup_memory=1` into `cmdline.txt` file and reboot. After that `docker stats` shows the memory usage of containers.

And that's it. Now I can start coping over docker compose files and data/configuration files of docker containers I have on now deprecated server (x86_64) and see how many of them run out-of-the-box on arm64 environment and how many of them need some new compatible docker images. A quick check reveals that I have currently about 27 containers running. Some of them are probably not used, so good chance to do a cleanup.



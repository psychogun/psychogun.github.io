---
layout: default
title: Installing Kali on a Raspberry Pi 3 Model B
parent: Linux
nav_order: 12
---
*WORK IN PROGRESS as of 2019-09-15*
# How to install Kali Linux on a Raspberry Pi 3 Model B
{: .no_toc }
This is how I installed Kali Linux on an ARM device, namely the Raspberry Pi 3 Model B. 

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}
---
## Getting started
Download Kali Linux for ARM architecture:
[https://www.offensive-security.com/kali-linux-arm-images/](https://www.offensive-security.com/kali-linux-arm-images/)
NB! These images have a default password of “toor” and may have pre-generated SSH host keys

Verify the SHA256:
```bash
zxcv:Downloads mitnick$ shasum -a 256 kali-linux-2019.3a-rpi3-nexmon.img.xz
```
Compare it to what is listed on the download site. 

Extract `kali-linux-2019.3a-rpi3-nexmon.img.xz`.
Fire up balenaEtcher (macOS) and "Select Image", "Select target" and press "Flash!".

Insert your SD card into your Raspberry Pi and hook it up to a monitor + keyboard.

### Prerequsites
* Raspberry Pi 3 Model B

## Emergency mode
If you happen to enter emergency mode when booting, you should modify `fstab`, after you enter the default root password `toor`.

```bash
nano /etc/fstab

#/dev/mmcblk0p2  / ext4 errors=remount-ro 0 1
/dev/mmcblk0p2  /  ext4 ro 0 1
```
This should mount the rootfs as read-only on the next reboot.

Reboot system using:
```bash
shutdown -rF now
```
Run `fsck -fy`, if the system hasn't already

Remount the rootfs as read-write using:
```bash
mount -o remount,rw /dev/mmcblk0p2
```

(IMPORTANT) Change /etc/fstab back to normal, thus:
```bash
/dev/mmcblk0p2  / ext4 errors=remount-ro 0 1
#/dev/mmcblk0p2  /  ext4 ro 0 1
```
If /etc/fstab doesn't get changed back to normal, then the system will always mount the rootfs as read-only.

Reboot.

## Change SSH host keys
```bash
root@kali:~ rm /etc/ssh/ssh_host_*
root@kali:~ dpkg-reconfigure openssh-server
root@kali:~ service ssh restart
```

## Change root password
```bash
root@kali:~ passwd root
```

## Expand installation 
All the available space on your SD card is not in use- view the disk space by issuing `df -h`:
```bash
root@kali:~# df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/root       4.5G  3.9G  284M  94% /
devtmpfs        459M     0  459M   0% /dev
tmpfs           464M     0  464M   0% /dev/shm
tmpfs           464M  660K  463M   1% /run
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           464M     0  464M   0% /sys/fs/cgroup
/dev/mmcblk0p1  122M   67M   55M  55% /boot
tmpfs            93M  4.0K   93M   1% /run/user/113
tmpfs            93M     0   93M   0% /run/user/0
root@kali:~# 
```
Show the usable space for your SD card:
```bash
root@kali:~# fdisk -l /dev/mmcblk0
Disk /dev/mmcblk0: 14.47 GiB, 15523119104 bytes, 30318592 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x08dd1e94

Device         Boot  Start      End  Sectors   Size Id Type
/dev/mmcblk0p1           1   250000   250000 122.1M  c W95 FAT32 (LBA)
/dev/mmcblk0p2      250001 30308863 30058863  14.3G 83 Linux
root@kali:~# 
```
Expand your installation to the size of the partition with `resize2fs`:
```bash
root@kali:~# resize2fs /dev/mmcblk0p2 
root@kali:~# df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        15G  3.9G  9.6G  29% /
devtmpfs        459M     0  459M   0% /dev
tmpfs           464M     0  464M   0% /dev/shm
tmpfs           464M  660K  463M   1% /run
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           464M     0  464M   0% /sys/fs/cgroup
/dev/mmcblk0p1  122M   67M   55M  55% /boot
tmpfs            93M  4.0K   93M   1% /run/user/113
tmpfs            93M     0   93M   0% /run/user/0
```
## Update and upgrade
Update:
```bash
root@kali:~# apt-get update
```
Upgrade:
```bash
root@kali:~# apt-get upgrade
```
Do a dist-upgrade:
```bash
root@kali:~# apt-get dist-upgrade
```
And use `apt autoremove` to remove packages that are no longer required.

## WiFi
Use `iwlist` to scan for wireless networks:
```bash
root@kali:~# iwlist wlan0 scan
```
ESSID is the name of the network.

Encrypt the WPA password for the ESSID you wish to connect to:
```bash
root@kali:~# wpa_passphrase "Pretty Fly For A 2.4GHz WiFi" Ultimate-P4ssw0rd > /etc/wpa_supplicant/wpa_supplicant.conf
network={
	ssid="Pretty Fly For A 2.4GHz WiFi"
	#psk="Ultimate-P4ssw0rd"
	psk=5213529740af2ecc21237b450cca7ef3a271e4ae55c5273d79867abbbaed75f5
}
```
Delete the commented-out plain text password from the file (`#psk="Ultimate-P4ssw0rd"`)
```bash
root@kali:~# nano /etc/wpa_supplicant/wpa_supplicant.conf 
```
Restart the `wpa_supplicant` process with `killall` and `wpa_supplicant` with `-i` to specify interface, and `-c` for configuration file (`-B` is for background process). Run `dhclient wlan0` for making it requesting an IP from the DHCP server:
```bash
root@kali:~# killall wpa_supplicant
root@kali:~# wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf 
root@kali:~# dhclient wlan0
```
Now our Raspberry Pi should be connected to the specific WiFi network!
```bash
root@kali:~# ifconfig wlan0
wlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.13.140  netmask 255.255.255.0  broadcast 192.168.13.255
        ether 9a:cd:4b:d9:74:a3  txqueuelen 1000  (Ethernet)
        RX packets 295  bytes 16425 (16.0 KiB)
        RX errors 0  dropped 64  overruns 0  frame 0
        TX packets 6  bytes 1324 (1.2 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

## Errors
### E: Sub-process /usr/bin/dpkg returned an error code (1)
Try to reconfigure the package database. Probably the database got corrupted while installing a package. Reconfiguring often fixes the problem:
```bash
root@kali:~# dpkg --configure -a
```
And then try to issue `apt-get upgrade`. If that does not work, this worked for me:
```bash
root@kali:~# apt clean
root@kali:~# apt --fix-broken install
root@kali:~# apt-get upgrade
```

## Acknowledgments
* [https://raspberrypi.stackexchange.com/questions/64095/unable-to-run-fsck-on-dev-mmcblk0p2/64097#64097](https://raspberrypi.stackexchange.com/questions/64095/unable-to-run-fsck-on-dev-mmcblk0p2/64097#64097)
* [https://docs.kali.org/kali-on-arm/install-kali-linux-arm-raspberry-pi](https://docs.kali.org/kali-on-arm/install-kali-linux-arm-raspberry-pi)
* [https://null-byte.wonderhowto.com/how-to/set-up-headless-raspberry-pi-hacking-platform-running-kali-linux-0176182/](https://null-byte.wonderhowto.com/how-to/set-up-headless-raspberry-pi-hacking-platform-running-kali-linux-0176182/)
* [https://itsfoss.com/dpkg-returned-an-error-code-1/](https://itsfoss.com/dpkg-returned-an-error-code-1/)
* [https://carmalou.com/how-to/2017/08/16/how-to-generate-passcode-for-raspberry-pi.html](https://carmalou.com/how-to/2017/08/16/how-to-generate-passcode-for-raspberry-pi.html)
* [https://www.raspberrypi.org/documentation/configuration/wireless/wireless-cli.md] (https://www.raspberrypi.org/documentation/configuration/wireless/wireless-cli.md)
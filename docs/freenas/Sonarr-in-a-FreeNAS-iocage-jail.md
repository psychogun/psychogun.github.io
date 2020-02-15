---
layout: default
title: How to install Sonarr in a FreeNAS iocage jail
parent: FreeNAS
nav_order: 14
---

# How to install Sonarr in a FreeNAS iocage jail
{: .no_toc }
This is how i installed a Sonarr, an independent fork of Sonarr reworked for automatically downloading movies via Usenet and BitTorrent, on FreeNAS in a iocage jail.

## Table of contents
{: .no_toc .text-delta }
1. TOC
{:toc}
---

## Getting started
These instructions will tell you how to add storage to an iocage jail, download, install and run Sonarr as a specific user, and integrate it with Deluge.


### Prerequisites
* Knowledge of SSH and how to navigate to your jail in FreeNAS
* FreeNAS 11.2 and knowledge of how to create a jail with shares and knowledge of UNIX folder and files permissions

## Udate and upgrade
Update and upgrade your iocage jail first:
```tcsh
root@Sonarr:/ # pkg upgrade && pkg update
```

## Install Sonarr
```bash
root@Sonarr:~ # pkg install sonarr
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
The following 44 package(s) will be affected (of 0 checked):

New packages to be INSTALLED:
	sonarr: 2.0.0.5322
	mediainfo: 19.09
	libzen: 19.09
	libmediainfo: 19.09
	tinyxml2: 7.1.0,1
	mono: 5.10.1.57_2
	ca_root_nss: 3.49.2
	python27: 2.7.17_1
	readline: 8.0.1
	indexinfo: 0.3.1
	libffi: 3.2.1_3
	gettext-runtime: 0.20.1
	py27-pillow: 6.2.2
	tk86: 8.6.10_1
	libXrender: 0.9.10_2
	libX11: 1.6.9,1
	libxcb: 1.13.1
	libXdmcp: 1.1.3
	xorgproto: 2019.2
	libXau: 1.0.9
	libxml2: 2.9.10
	libpthread-stubs: 0.4
	libXext: 1.3.4,1
	libXScrnSaver: 1.2.3_2
	libXft: 2.3.3
	fontconfig: 2.12.6,1
	expat: 2.2.8
	freetype2: 2.10.1
	tcl86: 8.6.10
	py27-tkinter: 2.7.17_6
	py27-setuptools: 41.4.0_1
	webp: 1.0.3_1
	tiff: 4.1.0
	jpeg-turbo: 2.0.3
	jbigkit: 2.1_1
	png: 1.6.37
	giflib: 5.2.1
	openjpeg: 2.3.1
	lcms2: 2.9
	py27-olefile: 0.46
	libinotify: 20180201_1
	curl: 7.67.0
	libnghttp2: 1.40.0
	sqlite3: 3.30.1

Number of packages to be installed: 44

The process will require 439 MiB more space.
107 MiB to be downloaded.

Proceed with this action? [y/N]: y
```

Set `sonarr_enable="YES"` in `/etc/rc.conf`, then start the Sonarr service by executing
```bash
root@Sonarr:/usr/local/etc/rc.d # service sonarr start
```

Make the user `sonarr` the owner of Sonarr's startup directory. This will enable you to update Sonarr from the GUI. 
```bash
root@Sonarr:/usr/local/share # chown -R sonarr sonarr/
```

## Configuring folders
Stop the Sonarr jail. 

Add two mount points, one folder for the TV Shows, the other folder for handling the Deluge transfer.

* Source: /mnt/Breaking/TV Shows
* Destination: /mnt/Breaking/iocage/jails/Sonarr/root/mnt/TV Shows

* Source: /mnt/Breaking/Torrents
* Destination: /mnt/Breaking/iocage/jails/Sonarr/root/mnt/Torrents

Under the installation, a `sonarr` user and group was created with a `uid` and `gid` of `351`:

```bash
(...)
===> Creating groups.
Creating group 'sonarr' with gid '351'.
===> Creating users
Creating user 'sonarr' with uid '351'.
(...)
```

You can change the uid:gid with this;
```bash
root@Sonarr:~ # id sonarr
uid=351(sonarr) gid=351(sonarr) groups=351(sonarr)
root@Sonarr:~ # pw usermod sonarr -u 923
root@Sonarr:~ # pw usermod sonarr -g 923
root@Sonarr:~ # id sonarr
uid=923(sonarr) gid=923(sonarr) groups=923(sonarr)
```
Replace ownage of `351` with user:group `sonarr`:
root@Sonarr:~ # find / -user 351 -exec chown -h sonarr {} \;
root@Sonarr:~ # find / -group 351 -exec chgrp -h sonarr {} \;

```bash
root@Sonarr:~ # getfacl /mnt/TV\ Shows/
# file: /mnt/TV Shows/
# owner: 1000
# group: 1002
        group:1006:r-x---a-R-c---:fd-----:allow
        group:1014:r-x---a-R-c---:fd-----:allow
    group:90000009:r-x---a-R-c---:fd-----:allow
            group@:rwxpDdaARWc---:fd-----:allow
            owner@:rwxpDdaARWcCo-:fd-----:allow
root@Sonarr:~ # pw groupadd warez -g 1002
root@Sonarr:~ # pw usermod sonarr -G warez

root@Sonarr:~ # groups sonarr
351 sonarr warez

root@Sonarr:/mnt/Torrents # getfacl Finished/
# file: Finished/
# owner: 1000
# group: 922
            group@:rwxp-daARWc---:fd----I:allow
            owner@:rwxpDdaARWcCo-:fd----I:allow
root@Sonarr:/mnt/Torrents # 
root@Sonarr:/mnt/Torrents # pw groupadd deluge -g 922
root@Sonarr:/mnt/Torrents # pw usermod sonarr -G sonarr,warez,deluge
```

## Authors
Mr. Johnson


## Acknowledgments
* [https://www.cyberciti.biz/faq/linux-change-user-group-uid-gid-for-all-owned-files/](https://www.cyberciti.biz/faq/linux-change-user-group-uid-gid-for-all-owned-files/)


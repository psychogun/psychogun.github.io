---
layout: default
title: How to install Radarr in a FreeNAS iocage jail
parent: FreeNAS
nav_order: 13
---

# How to install Radarr in a FreeNAS iocage jail
{: .no_toc }
This is how i installed a Radarr, an independent fork of Sonarr reworked for automatically downloading movies via Usenet and BitTorrent, on FreeNAS in a iocage jail.

<details open markdown="block">
  <summary>
   Table of contents
  </summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

{: .no_toc .text-delta }

---

## Getting started
These instructions will tell you how to add storage to an iocage jail, install and run Radarr and integrate Radarr with Deluge.

### Prerequisites
* Knowledge of SSH and how to navigate to your jail in FreeNAS
* FreeNAS 11.2 and knowledge of how to create a jail with shares and knowledge of UNIX folder and files permissions

### Udate and upgrade
Update and upgrade your iocage jail first:
```tcsh
root@Radarr:/ # pkg upgrade && pkg update
```

Update and upgrade your iocage jail first:
```tcsh
root@Radarr:/ # pkg upgrade && pkg update
```

---

## Install radarr
```bash
root@Radarr:~ # pkg install radarr
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
The following 44 package(s) will be affected (of 0 checked):

New packages to be INSTALLED:
	radarr: 0.2.0.1450
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

The process will require 440 MiB more space.
108 MiB to be downloaded.

Proceed with this action? [y/N]: y
(...)

=====
Message from mono-5.10.1.57_2:

--
If you have build/runtime errors with Mono and Gtk# apps please try the
following first:

* Build Mono and gtk+ (x11-toolkits/gtk20) without CPUTYPE and with the
  default FreeBSD CFLAGS ('-O2 -fno-strict-aliasing -pipe') as Mono has
  been known to expose compiler bugs.

* Try building and running Mono with the GENERIC kernel.
  - Mono requires SYSVSHM, SYSVMSG, and SYSVSEM which are part of the
    GENERIC kernel.
  - Removing kernel options or changing defaults to use experimental
    options can adversely affect Mono's ability to build and run.

* Remove leftover semaphores / increase semaphore limits.
  - Close apps which use Mono and run `ipcs -sbt`.  Remove the
    semaphores with MODE "--rw-------" and NSEMS "8" using ipcrm (1)
  - _OR_ simply reboot which is the safest method.
  - On multi-user systems the semaphore limits may need to be increased
    from the defaults. The following should comfortably support 30 users.

    # echo "kern.ipc.semmni=40" >> /boot/loader.conf
    # echo "kern.ipc.semmns=300" >> /boot/loader.conf

* If you are in a jailed environment, ensure System V IPC are enabled.
  You can rely on the security.jail.sysvipc_allowed  sysctl to check
  this status.  The following enables this feature on the host system:
    # echo "jail_sysvipc_allow=\"YES\"" >> /etc/rc.conf

* Some process information are accessed through /proc (e.g. when using
  NUnit) and procfs(5) has to be mounted for these features to work:
    # echo "proc            /proc   procfs  rw 0 0" >> /etc/fstab
root@Radarr:~ # 
```
To be able to update Radarr, the user which executes the service, (`radarr`), needs rights to the Startup directory:
```bash
root@Radarr:~ # cd /usr/local/share/
root@Radarr:/usr/local/share # chown -R radarr radarr/
```

Set `radarr_enable="YES"` in `/etc/rc.conf`, then start the Radarr service by executing
```bash
root@Radarr:/usr/local/etc/rc.d # service radarr start
```

---

## Configuring folders
Stop the Radarr jail. 

Add two mount points, one folder for the Movies, the other folder for handling the Deluge transfer.

* Source: /mnt/Breaking/Movies
* Destination: /mnt/Breaking/iocage/jails/Radarr/root/mnt/Movies

* Source: /mnt/Breaking/Torrents
* Destination: /mnt/Breaking/iocage/jails/Radarr/root/mnt/Torrents

```bash
root@Radarr:/mnt # getfacl Movies/
# file: Movies/
# owner: 1000
# group: 1002
            group@:rwxpDdaARWc---:fd-----:allow
        group:1006:r-x---a-R-c---:fd-----:allow
        group:1050:r-x---a-R-c---:fd-----:allow
        group:1052:rwxp-daARWc---:fd-----:allow
          user:922:rwxpD-aARWc---:fd-----:allow
            owner@:rwxpDdaARWcCo-:fd-----:allow
oot@Radarr:/mnt # pw groupadd movies -g 1052
root@Radarr:/mnt # pw usermod radarr -G movies
```

```bash
root@Radarr:/mnt # getfacl Torrents/
# file: Torrents/
# owner: 1000
# group: 922
            group@:rwxp-daARWc---:fd-----:allow
            owner@:rwxpDdaARWcCo-:fd-----:allow
root@Radarr:/mnt # pw groupadd deluge -g 922
root@Radarr:/mnt # pw usermod radarr -G movies,deluge
root@Radarr:/mnt # groups radarr
radarr movies deluge
```

---

## User configuration
```bash
===> Creating groups.
Creating group 'radarr' with gid '352'.
===> Creating users
Creating user 'radarr' with uid '352'.
```
Under the installation, a `radarr` user and group was created with a `uid` and `gid` of `352`:

You can change that users `uid` from `352` to for example `923` with this:
```bash
root@Radarr:~ # id radarr
uid=352(radarr) gid=352(radarr) groups=352(radarr)
root@Radarr:~ # pw usermod radarr -u 923
root@Radarr:~ # id radarr
uid=923(radarr) gid=351(radarr) groups=351(radarr)
```

Replace ownage of `352` with user:group `radarr`:
```bash
root@Radarr:~ # find / -user 352 -exec chown -h radarr {} \;
root@Radarr:~ # find / -group 352 -exec chgrp -h radarr {} \;
```

---

## Mono 5.20
```bash
pkg install screen
pkg install llvm90
pkg install -y llvm80 libepoxy-1.5.2
```


```bash
nano /tmp/mono-patch-5.20.1.34
```

Copy everything from [https://bz-attachments.freebsd.org/attachment.cgi?id=205999](https://bz-attachments.freebsd.org/attachment.cgi?id=205999) into that file, save and close

---

## Authors
Mr. Johnson

---

## Acknowledgments
* [https://github.com/Radarr/Radarr/wiki/Installation#freebsd-jail](https://github.com/Radarr/Radarr/wiki/Installation#freebsd-jail).
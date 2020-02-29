---
layout: default
title: How to install Radarr in a FreeNAS iocage jail
parent: FreeNAS
nav_order: 13
---

# How to install Radarr in a FreeNAS iocage jail
{: .no_toc }
This is how i installed a Radarr, an independent fork of Sonarr reworked for automatically downloading movies via Usenet and BitTorrent, on FreeNAS in a iocage jail.

## Table of contents
{: .no_toc .text-delta }
1. TOC
{:toc}
---

## Getting started
These instructions will tell you how to add storage to an iocage jail, install and run Radarr and integrate Radarr with Deluge.


### Prerequisites
* Knowledge of SSH and how to navigate to your jail in FreeNAS
* FreeNAS 11.2 and knowledge of how to create a jail with shares and knowledge of UNIX folder and files permissions

## Udate and upgrade
Update and upgrade your iocage jail first:
```tcsh
root@Airdccp:/ # pkg upgrade && pkg update
```

Update and upgrade your iocage jail first:
```tcsh
root@Airdccp:/ # pkg upgrade && pkg update
```

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
root@Radarr:~ # 
```

Set `radarr_enable="YES"` in `/etc/rc.conf`, then start the Radarr service by executing
```bash
root@Radarr:/usr/local/etc/rc.d # service radarr start
```


## User configuration
```bash
===> Creating groups.
Creating group 'radarr' with gid '352'.
===> Creating users
Creating user 'radarr' with uid '352'.
```

Message from freetype2-2.10.1:

--
The 2.7.x series now uses the new subpixel hinting mode (V40 port's option) as
the default, emulating a modern version of ClearType. This change inevitably
leads to different rendering results, and you might change port's options to
adapt it to your taste (or use the new "FREETYPE_PROPERTIES" environment
variable).

The environment variable "FREETYPE_PROPERTIES" can be used to control the
driver properties. Example:

FREETYPE_PROPERTIES=truetype:interpreter-version=35 \
	cff:no-stem-darkening=1 \
	autofitter:warping=1

This allows to select, say, the subpixel hinting mode at runtime for a given
application.

If LONG_PCF_NAMES port's option was enabled, the PCF family names may include
the foundry and information whether they contain wide characters. For example,
"Sony Fixed" or "Misc Fixed Wide", instead of "Fixed". This can be disabled at
run time with using pcf:no-long-family-names property, if needed. Example:

FREETYPE_PROPERTIES=pcf:no-long-family-names=1

How to recreate fontconfig cache with using such environment variable,
if needed:
# env FREETYPE_PROPERTIES=pcf:no-long-family-names=1 fc-cache -fsv

The controllable properties are listed in the section "Controlling FreeType
Modules" in the reference's table of contents
(/usr/local/share/doc/freetype2/reference/site/index.html, if documentation was installed).
=====
Message from python27-2.7.17_1:

--
Note that some standard Python modules are provided as separate ports
as they require additional dependencies. They are available as:

bsddb           databases/py-bsddb
gdbm            databases/py-gdbm
sqlite3         databases/py-sqlite3
tkinter         x11-toolkits/py-tkinter
--
===>   NOTICE:

This port is deprecated; you may wish to reconsider installing it:

EOLed upstream.

It is scheduled to be removed on or after 2020-12-31.
=====
Message from py27-setuptools-41.4.0_1:

--
Only /usr/local/bin/easy_install-2.7 script has been installed
  since Python 2.7 is not the default Python version.
=====
Message from ca_root_nss-3.49.2:

--
FreeBSD does not, and can not warrant that the certification authorities
whose certificates are included in this package have in any way been
audited for trustworthiness or RFC 3647 compliance.

Assessment and verification of trust is the complete responsibility of the
system administrator.


This package installs symlinks to support root certificates discovery by
default for software that uses OpenSSL.

This enables SSL Certificate Verification by client software without manual
intervention.

If you prefer to do this manually, replace the following symlinks with
either an empty file or your site-local certificate bundle.

  * /etc/ssl/cert.pem
  * /usr/local/etc/ssl/cert.pem
  * /usr/local/openssl/cert.pem
=====
Message from libinotify-20180201_1:

--
Libinotify functionality on FreeBSD is missing support for

  - detecting a file being moved into or out of a directory within the
    same filesystem
  - certain modifications to a symbolic link (rather than the
    file it points to.)

in addition to the known limitations on all platforms using kqueue(2)
where various open and close notifications are unimplemented.

This means the following regression tests will fail:

Directory notifications:
   IN_MOVED_FROM
   IN_MOVED_TO

Open/close notifications:
   IN_OPEN
   IN_CLOSE_NOWRITE
   IN_CLOSE_WRITE

Symbolic Link notifications:
   IN_DONT_FOLLOW
   IN_ATTRIB
   IN_MOVE_SELF
   IN_DELETE_SELF

Kernel patches to address the missing directory and symbolic link
notifications are available from:

https://github.com/libinotify-kqueue/libinotify-kqueue/tree/master/patches

You might want to consider increasing the kern.maxfiles tunable if you plan
to use this library for applications that need to monitor activity of a lot
of files.
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
root@Radarr:~ # 
root@Radarr:~ # 

## Configuring folders
Stop the Sonarr jail. 

Add two mount points, one folder for the TV Shows, the other folder for handling the Deluge transfer.

* Source: /mnt/Breaking/Movies
* Destination: /mnt/Breaking/iocage/jails/Radarr/root/mnt/Movies

* Source: /mnt/Breaking/Torrents
* Destination: /mnt/Breaking/iocage/jails/Radarr/root/mnt/Torrents

Under the installation, a `radarr` user and group was created with a `uid` and `gid` of `352`:


```bash
===> Creating groups.
Creating group 'radarr' with gid '352'.
===> Creating users
Creating user 'radarr' with uid '352'.
```


You can change it with this;
```bash
root@Sonarr:~ # id sonarr
uid=351(sonarr) gid=351(sonarr) groups=351(sonarr)
root@Sonarr:~ # pw usermod sonarr -u 923
root@Sonarr:~ # id sonarr
uid=923(sonarr) gid=351(sonarr) groups=351(sonarr)
```
Replace ownage of `351` with user:group `sonarr`:
root@Sonarr:~ # find / -user 351 -exec chown -h sonarr {} \;
root@Sonarr:~ # find / -group 351 -exec chgrp -h sonarr {} \;

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

## Authors
Mr. Johnson

## Acknowledgments
* [https://github.com/Radarr/Radarr/wiki/Installation#freebsd-jail](https://github.com/Radarr/Radarr/wiki/Installation#freebsd-jail).
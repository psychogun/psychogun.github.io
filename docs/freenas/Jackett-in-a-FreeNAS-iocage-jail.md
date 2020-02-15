---
layout: default
title: How to install Jackett in a FreeNAS iocage jail
parent: FreeNAS
nav_order: 6
---

# How to install Jackett in a FreeNAS iocage jail
{: .no_toc }
This is how i installed jackett, which works as a proxy server- it translates queries from apps (Sonarr, Radarr, SickRage, CouchPotato, Mylar, Lidarr, DuckieTV, qBittorrent, Nefarious etc.) into tracker-site-specific http queries, parses the html response, then sends results back to the requesting software. This allows for getting recent uploads (like RSS) and performing searches. Jackett is a single repository of maintained indexer scraping & translation logic - removing the burden from other apps.

## Table of contents
{: .no_toc .text-delta }
1. TOC
{:toc}
---

## Getting started
Show some love for Jackett over at [https://github.com/Jackett/Jackett](https://github.com/Jackett/Jackett).

These instructions will download, compile, install and run AirDC++ in an iocage jail on FreeNAS. There are no provided binaries for BSD on [https://airdcpp-web.github.io/docs/installation/installation.html](https://airdcpp-web.github.io/docs/installation/installation.html).

### Prerequisites
* Knowledge of SSH and how to navigate to your jail in FreeNAS
* FreeNAS 11.2 and knowledge of how to create a jail with shares and knowledge of UNIX folder and files permissions

## Installing

```bash
root@Jackett:~ # pkg update
root@Jackett:~ # pkg upgrade

Change from `quarterly` to `latest` in `FreeBSD.conf`:
```bash
root@Lidarr:~ # nano /etc/pkg/FreeBSD.conf
  url: "pkg+http://pkg.FreeBSD.org/${ABI}/latest",
```

Install Jackett:
```bash
root@Jackett:~ # pkg install jackett
Updating FreeBSD repository catalogue...
pkg: Repository FreeBSD has a wrong packagesite, need to re-create database
[Jackett] Fetching meta.txz: 100%    944 B   0.9kB/s    00:01    
[Jackett] Fetching packagesite.txz: 100%    6 MiB 283.3kB/s    00:23    
Processing entries: 100%
FreeBSD repository update completed. 31769 packages processed.
All repositories are up to date.
The following 37 package(s) will be affected (of 0 checked):

New packages to be INSTALLED:
	jackett: 0.12.1486
	mono: 5.10.1.57_2
	ca_root_nss: 3.49.1
	python27: 2.7.17_1
	readline: 8.0.1
	libffi: 3.2.1_3
	py27-pillow: 6.2.0
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
	py27-setuptools: 44.0.0
	webp: 1.1.0
	tiff: 4.1.0
	jpeg-turbo: 2.0.3
	jbigkit: 2.1_1
	png: 1.6.37
	giflib: 5.2.1
	openjpeg: 2.3.1
	lcms2: 2.9
	py27-olefile: 0.46
	libinotify: 20180201_1
	curl: 7.68.0
	libnghttp2: 1.40.0

Number of packages to be installed: 37

The process will require 415 MiB more space.
104 MiB to be downloaded.

Proceed with this action? [y/N]: y
```

```
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
root@Jackett:~ # 
```

To make Jackett start as a service on boot, enter the following command:
```bash
root@Jackett:~ # sysrc "jackett_enable=YES"
jackett_enable:  -> YES
```

.. or you could edit this file manually:
```bash
root@Jackett:~ # nano /etc/rc.conf
```

## Update Jackett
For some reason, the owner:group of the folder which `jackett` resides (`/usr/local/share/jackett`) was owned by a random UID:GID. If this is the case with you, you will be unable to update jackett automatically through the GUI. 

### Automatic update
Check which version of Jackett you are running by looking at the bottom in the web-gui. E.g.: Jackett Version 0.12.1632.0

If you have started Jackett, ensure `jackett` service is stopped by entering the following:
```bash
service jackett stop
```

Change ownership of the `/usr/local/share/jackett` folder.
```bash
root@Jackett:/usr/local/share # chown -R jackett:jackett jackett
```
Then start jackett again and click "Check for updates":
```bash
service jackett start
```
I now have successfully updated to Jackett Version 0.12.1815.0


### Manual update
If the above did not fix your problem, you can do a manual update by grabbing the latest version from the GitHub repository.

If you have started Jackett, ensure `jackett` service is stopped by entering the following:
```bash
service jackett stop
```

Next we will need to download the latest version of Jackett from its GitHub repository. First you will need to go to Jackett GitHub releases page and make a note of the latest version number. At the time of the writing this article, the latest version is v0.11.802. https://github.com/Jackett/Jackett/releases

Download the latest version with this command (replacing “v0.12.1632” with the latest version found above):
```bash
fetch https://github.com/Jackett/Jackett/releases/download/v0.12.1632/Jackett.Binaries.Mono.tar.gz -o /usr/local/share
```

Unzip the downloaded file:
```bash
root@Jackett:~ # tar -xzvf /usr/local/share/Jackett.Binaries.Mono.tar.gz -C /usr/local/share
```

Delete the downloaded `tar.gz` file:
```bash
root@Jackett:~ # rm /usr/local/share/Jackett.Binaries.Mono.tar.gz
```

Make a backup of the original `pkg` installation:
```bash
root@Jackett:~ # mv /usr/local/share/jackett /usr/local/share/jackett-old
```
`mv` our extracted Jackett binaries to original folder:
```bash
root@Jackett:~ # mv /usr/local/share/Jackett /usr/local/share/jackett
```

Then start jackett again
```bash
service jackett start
```


## Acknowledgments
* [https://digimoot.wordpress.com/2019/10/13/freenas-jackett-manual-install/](https://digimoot.wordpress.com/2019/10/13/freenas-jackett-manual-install/)
---
layout: default
title: How to install Plex in a FreeNAS iocage jail
parent: FreeNAS
nav_order: 11
---

# How to install Plex in a FreeNAS iocage jail
{: .no_toc }
This is how i installed a client-server media player system and software suite on FreeNAS in a standalone iocage jail. 

## Table of contents
{: .no_toc .text-delta }
1. TOC
{:toc}
---

## Getting started
I wrote this down because I moved from a warden jail on FreeNAS to an iocage jail due to an update broke my Plex on warden. Some of these settings might not apply for your installation. 

## Prerequisites
* FreeNAS 11.2
* Knowledge of SSH and stuff

## Create a jail
Go to your FreeNAS. Select Jails. Click ADD.

Jail Name: Plex
Release: 11.2-RELEASE

Click NEXT.

v DHCP Autconfigure IPv4

Click NEXT.

Jail Summary
Jail Name : Plex
Release : 11.3-RELEASE
DHCP Autoconfigure IPv4 : Yes
VNET Virtual Networking : Yes
Confirm these settings.

Click SUBMIT.

## Configure correct interface 
Go to your FreeNAS. Select Jails. Select 'Plex', click the three dots, and then click Edit.
Go to Network properties.
On Interfaces, type vnet0:bridge1. 

Click Save. 

## Start the jail
Go to your FreeNAS. Select Jails. Select 'Plex', click the three dots, and then click Start. 

## Log in to your jail
```bash
hubbard:~ jordan$ ssh -l grimes 192.168.200.177
grimes@freenas:~ # jls
   JID  IP Address      Hostname                      Path
    32                  Plex                 /mnt/williams/iocage/jails/Plex/root
grimes@freenas:~ # iocage console Plex
```

## Update & upgrade your jail
```bash
root@Plex:~ # env ASSUME_ALWAYS_YES=YES pkg bootstrap
Bootstrapping pkg from pkg+http://pkg.FreeBSD.org/FreeBSD:11:amd64/quarterly, please wait...
Verifying signature with trusted certificate pkg.freebsd.org.2013102301... done
[Plex] Installing pkg-1.12.0...
[Plex] Extracting pkg-1.12.0: 100%

root@Plex:~ # pkg update
Updating FreeBSD repository catalogue...
[Plex] Fetching meta.txz: 100%    944 B   0.9kB/s    00:01    
[Plex] Fetching packagesite.txz: 100%    6 MiB   2.2MB/s    00:03    
Processing entries: 100%
FreeBSD repository update completed. 32947 packages processed.
All repositories are up to date.

root@Plex:~ # pkg upgrade -y
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
Updating database digests format: 100%
Checking for upgrades (1 candidates): 100%
Processing candidates (1 candidates): 100%
Checking integrity... done (0 conflicting)
Your packages are up to date.
```

## Install plexmediaserver-plexpass (+ nano)
```bash
root@Plex:~ # pkg install -y nano plexmediaserver-plexpass

Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
The following 4 package(s) will be affected (of 0 checked):

New packages to be INSTALLED:
	nano: 4.4
	plexmediaserver-plexpass: 1.17.0.1709
	indexinfo: 0.3.1
	gettext-runtime: 0.20.1

Number of packages to be installed: 4

The process will require 194 MiB more space.
69 MiB to be downloaded.
[Plex] [1/4] Fetching nano-4.4.txz: 100%  509 KiB 521.3kB/s    00:01    
[Plex] [2/4] Fetching plexmediaserver-plexpass-1.17.0.1709.txz: 100%   69 MiB   7.2MB/s    00:10    
[Plex] [3/4] Fetching indexinfo-0.3.1.txz: 100%    6 KiB   5.8kB/s    00:01    
[Plex] [4/4] Fetching gettext-runtime-0.20.1.txz: 100%  151 KiB 154.5kB/s    00:01    
Checking integrity... done (0 conflicting)
[Plex] [1/4] Installing indexinfo-0.3.1...
[Plex] [1/4] Extracting indexinfo-0.3.1: 100%
[Plex] [2/4] Installing gettext-runtime-0.20.1...
[Plex] [2/4] Extracting gettext-runtime-0.20.1: 100%
[Plex] [3/4] Installing nano-4.4...
[Plex] [3/4] Extracting nano-4.4: 100%
[Plex] [4/4] Installing plexmediaserver-plexpass-1.17.0.1709...
===> Creating groups.
Creating group 'plex' with gid '972'.
===> Creating users
Creating user 'plex' with uid '972'.
[Plex] [4/4] Extracting plexmediaserver-plexpass-1.17.0.1709: 100%
=====
Message from plexmediaserver-plexpass-1.17.0.1709:

--
multimedia/plexmediaserver_plexpass includes an RC script:
/usr/local/etc/rc.d/plexmediaserver_plexpass

TO START PLEXMEDIASERVER ON BOOT:
sysrc plexmediaserver_plexpass_enable=YES

START MANUALLY:
service plexmediaserver_plexpass start

Once started, visit the following to configure:
http://localhost:32400/web

@@@ INTEL GPU OFFLOAD NOTES @@@

If you have a supported Intel GPU, you can leverage hardware
accellerated encoding/decoding in Plex Media Server on FreeBSD 12.0+.

The requirements are as follows:

* Install multimedia/drm-kmod: e.g., pkg install drm-fbsd12.0-kmod

* Enable loading of kernel module on boot: sysrc kld_list+="drm" 
** If Plex will run in a jail, you must load the module outside the jail!

* Load the kernel module now: kldload drm

* Install the supporting Intel VA support library for your GPU
** multimedia/libva-intel-driver: [LEGACY] Intel GMA 4500 or newer
** multimedia/libva-intel-media-driver: Intel HD 5000 (Gen8) or newer
*** This must be installed beside Plex. e.g., in the jail with Plex

* Add plex user to the video group: pw groupmod -n video -m plex

* For jails, make a devfs ruleset to expose /dev/dri/* devices.

e.g., /dev/devfs.rules on the host:

[plex_drm=10]
add include $devfsrules_hide_all
add include $devfsrules_unhide_basic
add include $devfsrules_unhide_login
add include $devfsrules_jail
add path 'dri*' unhide
add path 'dri/*' unhide
add path 'drm*' unhide
add path 'drm/*' unhide

* Enable the devfs ruleset for your jail. e.g., devfs_ruleset=10 in your
/etc/jail.conf or for iocage, iocage set devfs_ruleset="10" 

Please refer to documentation for all other FreeBSD jail management
utilities.

@@@ INTEL GPU OFFLOAD NOTES @@@

This is the PlexPass release channel of Plex. It is bleeding edge and
new features may not function without an active PlexPass account.
root@Plex:~ # exit
```

## Mount drives
Go to your FreeNAS. Select Jails. Select 'Plex', click the three dots, and then click Stop.
Select the three dots again and click Mount points.

Actions > + Add Mount Point

Source: /mnt/williams/Movies
Destination: /media/Movies
v Read only
Click Save

Source: /mnt/williams/TV Shows
Destination: /media/TV Shows
v Read only
Click Save

Source: /mnt/williams/Audio
Destination: /media/Audio
v Read only
Click Save

Go to your FreeNAS. Select Jails. Select 'Plex', click the three dots, and then click Start.

## Groups and rights
Go to your FreeNAS. Select Accounts > Groups. Click Add. I want to segregate permissions, so I am creating read only groups for the shares. 

GID: 3010
Name: movies_ro
uncheck Permit Sudo
uncheck Allow repeated GIDs

GID: 3040
Name: tv_shows_ro
uncheck Permit Sudo
uncheck Allow repeated GIDs

GID: 3060
Name: audio_ro
uncheck Permit Sudo
uncheck Allow repeated GIDs

Check which groups the user `plex` in our jail belongs to:
```bash
grimes@freenas:~ # jls
   JID  IP Address      Hostname                      Path
    33                  Plex                 /mnt/williams/iocage/jails/Plex/root
grimes@freenas:~ # iocage console Plex
root@Plex:~ # 
root@Plex:~ # groups plex
plex
root@Plex:~ # 
```

Add the same groups with the same GIDs you have made in your FreeNAS, inside your jail:
```bash
root@Plex:~ # pw groupadd movies_ro -g 3010
root@Plex:~ # pw groupadd tv_shows_ro -g 3040
root@Plex:~ # pw groupadd audio_ro -g 3060
```
Add `movies_ro`, `tv_shows_ro` and `audio_ro` to our user `plex`:
```bash
root@Plex:~ # pw usermod plex -G plex,audio_ro,tv_shows_ro,movies_ro
```
Check again which groups the user `plex` now belongs to:
```bash
root@Plex:~ # groups plex
plex audio_ro tv_shows_ro movies_ro
```
Now, our user `plex` belongs to groups that has read-only access on the different datasets, through Windows ACL. 

## Backup your Preferences.xml file
```bash
root@Plex:~# cd /usr/local/plexdata-plexpass/Plex\ Media\ Server/
root@Plex:/usr/local/plexdata-plexpass/Plex Media Server # mv Preferences.xml Preferences.xml.bak
```
## Copy Preferences.xml from the warden jail
```bash
root@Plex:~# exit
grimes@freenas:/ # cp /mnt/williams/jails/Plex/usr/local/plexdata-plexpass/Plex\ Media\ Server/Preferences.xml /mnt/williams/iocage/jails/Plex/root/usr/local/plexdata-plexpass/Plex\ Media\ Server/Preferences.xml
grimes@freenas:~ # iocage console Plex
root@Plex:/ # cd /usr/local/plexdata-plexpass/Plex\ Media\ Server/
root@Plex:/usr/local/plexdata-plexpass/Plex Media Server # chown plex Preferences.xml
```
(I did not bother copying over old cache and stuff).


## Start plexmediaserver_plexpass on boot
```bash
root@Plex:~ # sysrc plexmediaserver_plexpass_enable=YES
plexmediaserver_plexpass_enable:  -> YES
```

## Manual start of plexmediaserver_plexpass
```bash
root@Plex:/ # service plexmediaserver_plexpass start
Starting plexmediaserver_plexpass.
```

## Configure plex
```bash
root@Plex:~ # ifconfig
lo0: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> metric 0 mtu 16384
	options=600003<RXCSUM,TXCSUM,RXCSUM_IPV6,TXCSUM_IPV6>
	inet6 ::1 prefixlen 128 
	inet6 fe80::1%lo0 prefixlen 64 scopeid 0x1 
	inet 127.0.0.1 netmask 0xff000000 
	nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>
	groups: lo 
epair0b: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
	options=8<VLAN_MTU>
	ether ba:cc:60:34:12:bb
	hwaddr 01:17:11:08:11:0b
	inet 192.168.200.180 netmask 0xffffff00 broadcast 192.168.200.255 
	nd6 options=1<PERFORMNUD>
	media: Ethernet 10Gbase-T (10Gbase-T <full-duplex>)
	status: active
	groups: epair 
root@Plex:~ # 
```
Use a webbrowser and go to https://192.168.200.180:32400. There I only had to point the different libraries to my three folders in `/media`. 
PS: Remember to update your DHCP settings / and NAT settings for the new jail - or just shut down the warden jail and configure the iocage jail to be the same IP.  

## Automatic update 

If you dont want to have to bother executing the upgrades, you can add a crontab entry in the jail to do it for you:
```bash
0 4 * * * env ASSUME_ALWAYS_YES=YES pkg upgrade plexmediaserver >> ~/plexupgrade.log
```
(or for plex pass)
```bash
0 4 * * * env ASSUME_ALWAYS_YES=YES pkg upgrade plexmediaserver-plexpass >> ~/plexupgrade.log
```

## Manual update
```bash
root@Plex:/ # service plexmediaserver_plexpass stop
root@Plex:/ # pkg update && pkg upgrade 
root@Plex:/ # pkg upgrade multimedia/plexmediaserver-plexpass
root@Plex:/ # service plexmediaserver_plexpass start
```

## Troubleshooting
### Is it running?
```bash
root@Plex:/ # service plexmediaserver_plexpass status
plexmediaserver_plexpass is not running.
```

### What is the install status?
```bash
root@Plex:/ # pkg info plexmediaserver-plexpass
plexmediaserver-plexpass-1.17.0.1841
Name           : plexmediaserver-plexpass
Version        : 1.17.0.1841
Installed on   : Tue Oct  8 22:17:42 2019 CEST
Origin         : multimedia/plexmediaserver-plexpass
Architecture   : FreeBSD:11:amd64
Prefix         : /usr/local
Categories     : multimedia
Licenses       : 
Maintainer     : feld@FreeBSD.org
WWW            : https://plex.tv
Comment        : Plex Media Server component
Options        :
	RELAY          : on
Annotations    :
	FreeBSD_version: 1102000
	cpe            : cpe:2.3:a:plex:plex_media_server:1.17.0:::::freebsd11:x64
	no_provide_shlib: yes
	repo_type      : binary
	repository     : FreeBSD
Flat size      : 191MiB
Description    :
Plex Media Server is used to host the content and plugins that are then
streamed to Plex Media Center and Plex mobile app clients, either on the
same machine, the same local area network, or over the Internet. Content
may be transcoded by the server before it's streamed in order to reduce
bandwidth requirements, or for compatibility with the device being
streamed to.

WWW: https://plex.tv
```

### Check the logs
```bash
root@Plex:/usr/local/plexdata-plexpass/Plex Media Server # cd Logs
root@Plex:/usr/local/plexdata-plexpass/Plex Media Server/Logs # ls -alh
total 3255
drwxr-xr-x   3 plex  plex    47B Oct  8 22:09 .
drwxr-xr-x  10 plex  plex    11B Oct  8 20:31 ..
-rw-r--r--   1 plex  plex     0B Mar 22  2018 Plex DLNA Server Neptune.log
-rw-r--r--   1 plex  plex   2.6K Mar 22  2018 Plex DLNA Server.log
-rw-r--r--   1 plex  plex   9.4K Oct  8 07:04 Plex Media Scanner Analysis.log
-rw-r--r--   1 plex  plex    53K Oct  8 07:05 Plex Media Scanner Chapter Thumbnails.log
-rw-r--r--   1 plex  plex    15K Oct  8 07:08 Plex Media Scanner Deep Analysis.log
-rw-r--r--   1 plex  plex   1.6M Oct  8 18:05 Plex Media Scanner.log
-rw-r--r--   1 plex  plex   4.1M Oct  8 22:16 Plex Media Server.log
-rwxr-xr-x   1 plex  plex    13K Oct  8 22:09 Plex Transcoder Statistics.log
-rw-r--r--   1 plex  plex   1.6K Sep 11 20:31 Plex Tuner Service.log
drwxr-xr-x   2 plex  plex    81B Oct  8 18:05 PMS Plugin Logs

root@Plex:/usr/local/plexdata-plexpass/Plex Media Server/Logs # tail -f Plex\ Media\ Server.log 
```

### Check if our rc.d is installed:
```bash
root@Plex:/usr/local/plexdata-plexpass/Plex Media Server/Logs # service -e
/etc/rc.d/cleanvar
/etc/rc.d/netif
/etc/rc.d/newsyslog
/etc/rc.d/syslogd
/etc/rc.d/virecover
/etc/rc.d/motd
/usr/local/etc/rc.d/plexmediaserver_plexpass
/etc/rc.d/cron
```

### Check rc.d
```bash
root@Plex:/usr/local/plexdata-plexpass/Plex Media Server/Logs # more /usr/local/etc/rc.d/plexmediaserver_plexpass
```

### Verbose running 
```bash
root@Plex:/usr/local/plexdata-plexpass/Plex Media Server/Logs # sh -x /usr/local/etc/rc.d/plexmediaserver_plexpass start
```
## Authors
Mr. Johnson

## Acknowledgments
* [https://www.ixsystems.com/community/threads/how-to-update-freenas-11-2-plex-server-from-1-13-2-5154-to-1-13-5-5291.69059/page-2](https://www.ixsystems.com/community/threads/how-to-update-freenas-11-2-plex-server-from-1-13-2-5154-to-1-13-5-5291.69059/page-2)
* [https://support.plex.tv/articles/201370363-move-an-install-to-another-system/](https://support.plex.tv/articles/201370363-move-an-install-to-another-system/)
* [https://www.ixsystems.com/community/threads/step-by-step-to-install-plex-transmission-inside-iocage-jail-in-freenas-11-2-beta3.70617/](https://www.ixsystems.com/community/threads/step-by-step-to-install-plex-transmission-inside-iocage-jail-in-freenas-11-2-beta3.70617/)
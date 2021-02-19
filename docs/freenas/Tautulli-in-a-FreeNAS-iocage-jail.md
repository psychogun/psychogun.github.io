---
layout: default
title: How to install Tautulli server in a FreeNAS iocage jail
parent: FreeNAS
nav_order: 17
---

# How to install Tautulli in a FreeNAS iocage jail
{: .no_toc }
I wanted to access the Internet safely and securely from my smartphone or laptop when I am connected to an untrusted network such as the WiFi of a hotel or coffee shop. A Virtual Private Network (VPN) allows you to traverse untrusted networks privately and securely as if you were on a private network. The traffic emerges from the VPN server and continues its journey to the destination. 

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


### pkg update && pkg upgrade
```bash
root@Tautulli:~ # pkg update && pkg upgrade
The package management tool is not yet installed on your system.
Do you want to fetch and install it now? [y/N]: yes
```


### nano
```bash
root@Tautulli:~ # pkg install nano
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
The following 3 package(s) will be affected (of 0 checked):

New packages to be INSTALLED:
	nano: 4.6
	indexinfo: 0.3.1
	gettext-runtime: 0.20.1

Number of packages to be installed: 3

The process will require 3 MiB more space.
686 KiB to be downloaded.

Proceed with this action? [y/N]: y
```
### Change from quarterly to latest
To receive the latest updates from the packet manager, change this:
```bash
root@ELK:~ # nano /etc/pkg/FreeBSD.conf 
```
Change
```bash
FreeBSD: {
  url: "pkg+http://pkg.FreeBSD.org/${ABI}/quarterly",
```
to
```bash
FreeBSD: {
  url: "pkg+http://pkg.FreeBSD.org/${ABI}/latest",
```

---

## Install Tautulli
Change directory to `/usr/local/share`.

```bash
root@Tautulli:~ # pkg install tautulli
```

---

## Copy old database
Copy the old database to Tautulli's installation folder which is `/usr/local/www/tautulli/`.

```bash
root@freenas:/mnt/Baldhead/jails/Tautulli/usr/share # cp /mnt/Baldhead/jails/Tautulli/usr/local/share/Tautulli/tautulli.db /mnt/Baldhead/iocage/jails/Tautulli/root/var/db/tautulli/
```

---

## Enable autostart
```bash
root@Tautulli:/usr/local/www/tautulli # nano /etc/rc.conf
# Enable Tautulli     
tautulli_enable="YES"
```

---

## Fault finding
### rc.d/tautulli
The startup script is located here:
```bash
root@Tautulli:/usr/local/www/tautulli # more /usr/local/etc/rc.d/tautulli
#!/bin/sh
# Created by: Mark Felder <feld@FreeBSD.org>
#
# $FreeBSD: head/multimedia/tautulli/files/tautulli.in 466727 2018-04-07 14:14:23Z feld $
#
# PROVIDE: tautulli
# REQUIRE: LOGIN
# KEYWORD: shutdown
#
# Add the following lines to /etc/rc.conf to enable Tautulli:
#
# tautulli_enable="YES"
#

. /etc/rc.subr

name=tautulli
rcvar=tautulli_enable
load_rc_config $name

: ${tautulli_enable:=NO}
: ${tautulli_user=tautulli}
: ${tautulli_group=tautulli}

command_interpreter=/usr/local/bin/python2.7
command=/usr/local/www/tautulli/Tautulli.py
command_args="-d --nolaunch --datadir /var/db/tautulli"
start_precmd=tautulli_prestart

tautulli_prestart()
{
        if ! [ -e /etc/localtime ] ; then
                echo "Tautulli needs the system timezone to be set."
                echo "Please run /usr/sbin/tzsetup"
                exit 1
        fi
        
        install -d -o ${tautulli_user} -g ${tautulli_group} /var/db/tautulli
}

run_rc_command "$1"
```



Create a user for this service. 

```
Type: cd /usr/local/share/
Type: git clone https://github.com/Tautulli/Tautulli.gi
```

----

## Authors
Mr. Johnson

---

## Acknowledgments
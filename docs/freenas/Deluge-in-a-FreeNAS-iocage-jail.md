---
layout: default
title: How to install Deluge in a FreeNAS iocage jail
parent: FreeNAS
nav_order: 6
---

# How to install Deluge in a FreeNAS iocage jail
{: .no_toc }
From the very beginning I started to rip my CDs to be able to play them through a device that did not have a CD-ROM, both my naming scheme and idv3 tagging had been done in many various ways. This has now resulted in a very chaotic library, at least when accessed through Plex (albums and songs are literally populated from every album you could think of, due to plex default ommits the music library files' idv3 tag - making it use the files defaults, did no better). So i googled to look for a way to automate my naming scheme and to have a proper id tagging of my music files and I found Lidarr. 

Thiis is how i installed a Radarr, a music collection manager for Usenet and BitTorrent users, on FreeNAS in a iocage jail.

## Table of contents
{: .no_toc .text-delta }
1. TOC
{:toc}
---
## Getting started
Go to your FreeNAS. Select Jails > and click Add.
Jail Name: Deluge
Release: 11.2-RELEASE

Click NEXT.

v DHCP Autoconfigure IPv4
v VNET

Click NEXT.

Click SUBMIT.

Go to your FreeNAS. Select the three dots on the right for our newly created jail called Lidarr > Edit.
Network properties > Interfaces: vnet0:bridge1.

Click SAVE.

Select and START Lidarr.

## Install Deluge from pkg
### Log in to your FreeNAS via SSH
```bash
bram:~ cohen$ ssh -l ross 192.168.200.133
ross@freenas:~ # 
```
### List your jails
```bash
ross@freenas:~ # jls
   JID  IP Address      Hostname                      Path
    22                  Deluge                        /mnt/Interstella/iocage/jails/Deluge/root
```
### Log into your jail
```bash
ross@freenas:~ # iocage console Deluge
```
### Update and upgrade your jail
```bash
ross@Deluge:~ # pkg upgrade && pkg update
```
### User deluge
In the FreeNAS gui, go to Accounts > Users and select ADD.
Full Name: deluge service user
Username: deluge
Password: ***
Confirm password: ***

User ID: 922
uncheck New Primary Group
Primary Group: deluge

Home Directory: /nonexistent
Enable password login: no

Everything else is default.

Click SAVE.
### Create a deluge user within the jail
Be consistent with the UID (922) of the former created deluge user in FreeNAS gui. 
```bash
ross@Deluge:~ #  pw useradd -n deluge -u 922 -m -s /usr/sbin/nologin
```
With the command above, the default group for our user `deluge` will be a group called `deluge`, with GID=922. 
You can see for yourself, which groups that are created in your jail by typing in `more /etc/group`.

To see which groups `deluge` user is member of, write `groups deluge`. 

### pkg-install
```bash
ross@Deluge:~ #  pkg install deluge
Number of packages to be installed: 172

The process will require 2 GiB more space.
238 MiB to be downloaded.

Proceed with this action? [y/N]: y
```
### Data-dir
Create directories for deluge services to startup
```bash
ross@Deluge:~ # cd /home/deluge/
ross@Deluge:/home/deluge # mkdir .config
ross@Deluge:/home/deluge # cd .config
ross@Deluge:/home/deluge/.config # mkdir deluge 
ross@Deluge:~ # chown -R deluge:deluge /home/deluge/.config
```

### Autostart deluge
Add the following to /etc/rc.conf
```bash
ross@Deluge:~ # printf '\n# Enable Deluge\ndeluged_enable="YES"\ndeluge_web_enable="YES"' >> /etc/rc.conf
```
### Edit the startup script
Edit the `deluged` (deluge daemon) startup script to use the newly created deluge user.
```bash
ross@Deluge:~ # nano /usr/local/etc/rc.d/deluge_web
```
Locate the following line and update it from:

`: ${deluged_user:="asjklasdfjklasdf"}`
to
`: ${deluged_user:="deluge"}`

and change in `/usr/local/etc/rc.d/deluge_web` as well: 
`: ${deluge_web_user:="asjklasdfjklasdf"}`
to 
`: ${deluge_web_user:="deluge"}`

### Start 
```bash
ross@Deluge:/home/deluge # service deluged start
Starting deluged.
ross@Deluge:/home/deluge # service deluge_web start
Starting deluge_web.
ross@Deluge:/home/deluge # 
```

## Installing from git
Download the latest tarball: [https://ftp.osuosl.org/pub/deluge/source/2.0/](https://ftp.osuosl.org/pub/deluge/source/2.0/)
```bash
ross@Deluge:/home/deluge # wget https://ftp.osuosl.org/pub/deluge/source/2.0/deluge-2.0.3.tar.xz
```

Extract the tarball:
```bash
ross@Deluge:/home/deluge # tar -xzvf deluge-2.0.3.tar.xz
```

Change directory:
```bash
ross@Deluge:/home/deluge # cd deluge-2.0.3
```

Install python
pkg install python
Install Pip from py36-pip package:








pw useradd -n deluge -u 922 -d /nonexistent -s /usr/sbin/nologin




## Acknowledgments
https://blog.tkrn.io/deluge-in-a-freenas-11-iocage-jail-easy-tutorial/
* [https://www.freshports.org/net-p2p/deluge/](https://www.freshports.org/net-p2p/deluge/)
* [https://github.com/deluge-torrent/deluge/blob/develop/DEPENDS.md](https://github.com/deluge-torrent/deluge/blob/develop/DEPENDS.md)
* [https://computingforgeeks.com/how-to-install-pip-python-package-manager-on-freebsd-12/](https://computingforgeeks.com/how-to-install-pip-python-package-manager-on-freebsd-12/)
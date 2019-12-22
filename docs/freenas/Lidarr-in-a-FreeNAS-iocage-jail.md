---
layout: default
title: How to install Lidarr in a FreeNAS iocage jail
parent: FreeNAS
nav_order: 5
---

# How to install Lidarr in a FreeNAS iocage jail
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
Jail Name: Lidarr
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

## Install Lidarr
### Log in to your FreeNAS via SSH
```bash
bangalter:~ thomas$ ssh -l homem-christo 192.168.200.133
homem-christo@freenas:~ # 
```
### List your jails
```bash
homem-christo@freenas:~ # jls
   JID  IP Address      Hostname                      Path
    40                  Lidarr                        /mnt/Interstella/iocage/jails/Lidarr/root
```
### Log into your jail
```bash
homem-christo@freenas:~ # iocage console Lidarr
```
### Update and upgrade your jail
```bash
homem-christo@Lidarr:~ # pkg upgrade && pkg update
```
### Install Mono
```bash
homem-christo@Lidarr:~ # pkg install mono
The following 36 package(s) will be affected (of 0 checked):
New packages to be INSTALLED:
	mono: 5.10.1.57_2
    (...)
Proceed with this action? [y/N]: y
```
### Install sqlite3. nano and wget
```bash
homem-christo@Lidarr:/usr/local # pkg install sqlite3 nano wget
```
### Download Lidarr
Check for latest releases here: [https://github.com/lidarr/Lidarr/releases](https://github.com/lidarr/Lidarr/releases)
```bash
homem-christo@Lidarr:~ # cd /usr/local
homem-christo@Lidarr:/usr/local # wget https://github.com/lidarr/Lidarr/releases/download/v0.7.1.1381/Lidarr.master.0.7.1.1381.linux.tar.gz
```
### Extract Lidarr
```bash
homem-christo@Lidarr:/usr/local # tar -xzvf Lidarr.master.0.7.1.1381.linux.tar.gz
```
Remove the tarball by `rm Lidarr.master.0.7.1.1381.linux.tar.gz`.
### Try to run Lidarr
```bash
homem-christo@Lidarr:/usr/local # mono --debug /usr/local/Lidarr/Lidarr.exe
```
Lidarr should be available on port 8686, for example [http://localhost:8686](http://localhost:8686).

Close the running Lidarr instance with Ctrl + C
## Permissions
### Group lidarr
In the FreeNAS gui, go to Accounts > Groups and select ADD.
GID: 926
Name: lidarr
uncheck Permit Sudo
uncheck Allow repeated GIDs

Click SAVE.

### User lidarr
In the FreeNAS gui, go to Accounts > Users and select ADD.
Full Name: lidarr service user
Username: lidarr
Password: ***
Confirm password: ***

User ID: 926
uncheck New Primary Group
Primary Group: lidarr

Home Directory: /nonexistent
Enable password login: no

Everything else is default.

Click SAVE.

### Check permissions on the dataset
Use a Windows based computer > Win 7.
Go to Computer > Map network drive.

Mount your share, e.g. on disk Z: with \\freenas\audio with a user that is owner of the share.
Select the Z:\ drive and select Properties.
Go to the Security tab and click Edit...

Click Add...

Enter the object names to select: FREENAS\lidarr and click OK.

Make sure that the newly added group in your Security tab of the audio share has "Modify" and "Write" permissions a (check both "Modify" and "Write" under Allow). Click OK.

## Mount points
Stop your newly created Lidarr jail from the FreeNAS gui; FreeNAS > Jails > select Lidarr and click STOP.
Click the three dots on the right and select Mount Points, ACTIONS > + Add Mount Point.

Source: /mnt/Interstella/Audio
Destination: /mnt/Audio
uncheck Read-Only.

Start your Lidarr jail by selecting it in the FreeNAS gui and press START.
Log on to your jail again via SSH:
```bash
homem-christo@freenas:~ # iocage console Lidarr
```
## Create a lidarr user within the jail
Be consistent with the UID (926) of the former created user in FreeNAS gui.
```bash
homem-christo@Lidarr:/usr/local # pw useradd -n lidarr -u 926 -d /nonexistent -s /usr/sbin/nologin
homem-christo@Lidarr:/usr/local # pw useradd -n lidarr -u 926 -m -s /usr/sbin/nologin
```
## Create the daemon
Create the daemon so Lidarr will be able to start as a service.
```bash
homem-christo@Lidarr:/usr/local # cd /etc/rc.d/
homem-christo@Lidarr:/etc/rc.d # nano lidarr
#!/bin/sh
#
# Author: Jarod Sams
#

# PROVIDE: lidarr
# REQUIRE: LOGIN
# KEYWORD: shutdown

# Add the following lines to /etc/rc.conf to enable lidarr:
# lidarr_enable="YES"

. /etc/rc.subr

name="lidarr"
rcvar=lidarr_enable

load_rc_config $name

: ${lidarr_enable="NO"}
: ${lidarr_user:="lidarr"}
# This next directory can be changed to whatever you want
: ${lidarr__data_dir:="/usr/local/Lidarr/lidarr"}

pidfile="${lidarr__data_dir}/lidarr.pid"
# You may need to adjust the mono location if your mono executable is somewhere else
procname="/usr/local/bin/mono"
command="/usr/sbin/daemon"
# The directory laid out in the next line will need to be changed if your Lidarr directory is elsewhere
command_args="-f ${procname} /usr/local/Lidarr/Lidarr.exe --nobrowser --data=${lidarr__data_dir}"    
start_precmd=lidarr_precmd

lidarr_precmd()
{
	export XDG_CONFIG_HOME=${lidarr__data_dir}

	if [ ! -d ${lidarr__data_dir} ]; then
		install -d -o ${lidarr_user} ${lidarr__data_dir}
	fi
}

run_rc_command "$1"
```
Once you've got your lidarr file created and filled out, we need to make sure the ownership and permissions are correct and verify:
```bash
homem-christo@Lidarr:/etc/rc.d # chmod +x lidarr && ls -Fl lidarr
-rwxr-xr-x  1 root  wheel  1006 Oct 20 21:32 lidarr*
```
Now, add your new rc.d script to the startup
```bash
homem-christo@Lidarr:/etc/rc.d # echo 'lidarr_enable="YES"' >> /etc/rc.conf
```
Start the service
```bash
homem-christo@Lidarr:/etc/rc.d # service lidarr start
```
Navigate to your new Lidarr service in your browser to get started with configuration of Lidarr, e.g. http://ip_addr:8686

## Configuration of Lidarr

### Settings
#### Media Management
Track Naming
v Rename tracks
v Replace Illegal Characters
Standard Track Format:
{Artist Name} - {Album Title} [{Release Year}] - {track:00} {Track Title}
Multi Disc Track Format:
{Medium Format} {medium:00}/{track:00} {Artist Name} - {Album Title} [{Release Year}] - {Track Title}

Artist Folder Format
{Artist Name}

Album Folder Format
{Artist Name} - {Album Title} [{Release Year}]

Folders:
uncheck Create empty artist folders
uncheck Delete empty folders

Importing:
uncheck Skip Free Space Check
uncheck Use Hardlinks instead of Copy

uncheck Import Extra Files (I do not want to bring along *.nfo, *.m3u files and such)

File Management
uncheck Ignore Deleted Tracks
Propers and Repacks - Prefer and Upgrade
Rescan Artist Folder after Research
Always

Allow Fingerprinting - Always
Change File Date - None

Recycling Bin - 
Recycling Bin Cleanup - 

Permissions:
v Set permissions
File chmod mode 0644
Folder chmod mode 0755
chown User 1000
chown Group 1002

#### Profiles
Click on the 'Standard' Metadata Profiles and select everything. This allowed me to be able to find most of the ripped CDs and what nots I have ripped since day 0.

### Library
#### Import
What you first want to do, is to go through your collection of albums and artist, and make a single folder for every artist that you have.
Then move all your music / albums and such to the corresponding artist' folder.

After you have done that initially, you can set a 'Root folder'. Click 'Start Import' and select the root of the folder containing all the artists folders.

Lidarr will try to automatically choose the Artist for the Folder. Go through all the listed folders and see if they correspond with the Artist suggestions.


Settings > Metadata > Write Metadata to Audio Files
Tag Audio Files with Metadata: All files; keep in sync with MusicBrainz. 
Scrub Existing Tags: check.

Settings > Media Management
Rename Tracks V
Replace Illegal Characters V
Standard Track Format: {track:00} {Artist Name} - {Album Title} [{Release Year}] - {Track Title}
Multi Disck Track Format: {Medium Format} {medium:00}/{track:00} {Artist Name} - {Album Title} [{Release Year}] - {Track Title}
Album Folder Format: {Artist Name} - {Album Title} [{Release Year}]

Uncheck Use Hardlinks instead of Copy
Import Extra Files V

chown User 10000

#### Download Clients
Select +, press Deluge. 

Name: Deluge
v Enable
Host 
Port 8112
Password *****
Category: lidarr
Recent Priority Last
Older priority Last

Test, and if everything is OK - Save.

Go to Deluge and Right click on the Label tab to the left and add 'lidarr' without the quotes. Click OK.
Right-click on the label named lidarr and select Label Options. 

Rights. So, again. Which group is owner of the file to where Deluge moves the finished files?
We want to add our lidarr user to a group that has read+write permission to that folder.
So go through the route explained above, on Windows.
Or better yet, do a `getfacl` on the specific folder:
```bash
homem-christo@freenas:~ # getfacl /mnt/Interstella/Torrents/
# file: /mnt/Kaneda/Torrents/
# owner: homem-christo
# group: deluge
            group@:rwxp-daARWc---:fd-----:allow
            owner@:rwxpDdaARWcCo-:fd-----:allow
homem-christo@freenas:~ # 
```
So a group called deluge is group owner of that folder.

Check in FreeNAS gui which GID it has > Accounts > Groups > deluge. 
I have the group deluge with GID 922, so I would want to create a group inside of my Lidarr jail with GID deluge:
```bash
homem-christo@Lidarr:~ # groups lidarr
lidarr audio
homem-christo@Lidarr:~ # pw groupadd deluge -g 922
homem-christo@Lidarr:~ # groups lidarr
lidarr audio
homem-christo@Lidarr:~ # pw usermod lidarr -G lidarr,audio,deluge
homem-christo@Lidarr:~ # groups lidarr
lidarr audio deluge
homem-christo@Lidarr:~ # 
```

Then I will go on and stop the Lidarr jail, then click the three dots on the right > Mount Points > ACTIONS > Add Mount Point:
Source: /mnt/Interstella/Torrents
Destination: /mnt/Torrents
uncheck Read-Only

Click SAVE.

Start your Lidarr jail again.


#### Indexers
Select +, press Torsznab.

Name: Torsznab - RARBG
v Enable RSS
v Enable Automatic Search
v Enable Interactive Search
URL: 
API Path: /api
API Key: 
Categories: 100025 ,100023
Minimum Seeders: 5

## Fault finding
System > Status

### fpcalc could not be found. Audio fingerprinting disabled.

```bash
root@freenas:~ # iocage console Lidarr
root@Lidarr:~ # service lidarr stop
Stopping lidarr.
Waiting for PIDS: 74622, 74622, 74622.
root@Lidarr:~ # pkg install chromaprint
root@Lidarr:~ # service lidarr start
Starting lidarr.
root@Lidarr:~ # 
```

### Updating will not be possible to prevent deleting AppData on Update
[https://github.com/Lidarr/Lidarr/wiki/Health-checks#updating-will-not-be-possible-to-prevent-deleting-appdata-on-update](https://github.com/Lidarr/Lidarr/wiki/Health-checks#updating-will-not-be-possible-to-prevent-deleting-appdata-on-update)

App-Data directory must reside in another folder other than Lidarr's startup folder.
```bash
root@freenas:~ # iocage console Lidarr
root@Lidarr:/ # cd /usr/local/
root@Lidarr:/usr/local # service lidarr stop
root@Lidarr:/usr/local # mkdir Lidarr-AppData
root@Lidarr:/usr/local # cp -ri /usr/local/Lidarr/lidarr/ /usr/local/Lidarr-AppData/
root@Lidarr:/usr/local # chown -R lidarr /usr/local/Lidarr-AppData/
root@Lidarr:/usr/local # nano /etc/rc.d/lidarr 
```
Change `: ${lidarr__data_dir:="/usr/local/Lidarr/lidarr"}` to `: ${lidarr__data_dir:="/usr/local/Lidarr-AppData"}`

```bash
root@Lidarr:/usr/local # service lidarr start
```

### Currently installed Mono version 5.10.1.57 is supported but has some known issues. Please upgrade Mono to version 5.20.
[https://www.ixsystems.com/community/resources/how-to-manually-upgrade-mono-from-5-10-to-5-20-in-a-freenas-jail.126/](https://www.ixsystems.com/community/resources/how-to-manually-upgrade-mono-from-5-10-to-5-20-in-a-freenas-jail.126/)

```bash
root@freenas:~ # iocage console Lidarr
root@Lidarr:~ # service lidarr stop
```
Change from `quarterly` to `latest` in `FreeBSD.conf`:
```bash
root@Lidarr:~ # nano /etc/pkg/FreeBSD.conf
  url: "pkg+http://pkg.FreeBSD.org/${ABI}/latest",
```

```bash
root@Lidarr:~ # portsnap fetch extract
root@Lidarr:~ # pkg update
root@Lidarr:~ # pkg upgrade 
```
Copy everything from [https://bz-attachments.freebsd.org/attachment.cgi?id=205999](https://bz-attachments.freebsd.org/attachment.cgi?id=205999) to `/tmp/mono-patch-5.20.1.34`:
```bash
root@Lidarr:~ # nano /tmp/mono-patch-5.20.1.34

root@Lidarr:/usr/ports/lang/mono # 
root@Lidarr:/usr/ports/lang/mono # patch -E < /tmp/mono-patch-5.20.1.34
```
Change `PORTVERSION=` from `5.10.1.57` to `5.20.1.34`, then save and exit. It is right at the top few lines of `Makefile`:
```bash
root@Lidarr:/usr/ports/lang/mono # nano Makefile
```
Install necessary things to compile `Mono`:
```bash
root@Lidarr:/usr/ports/lang/mono # pkg install -y llvm80 libepoxy-1.5.2 perl5 mesa-dri
```
```bash
root@Lidarr:/usr/ports/lang/mono # nano /etc/make.conf
ALLOW_UNSUPPORTED_SYSTEM=yes
```
`pkg delete -f "*proto"``

Make install clean:
```bash
root@Lidarr:/usr/ports/lang/mono # make -DBATCH install clean
```
Just press 'Enter' if given any promts with options when issuing `deinstall`/`reinstall`:
```bash
root@Lidarr:/usr/ports/lang/mono # make deinstall reinstall
root@Lidarr:/usr/ports/lang/mono # pkg update
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
root@Lidarr:/usr/ports/lang/mono # pkg upgrade
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
Updating database digests format: 100%
Checking for upgrades (89 candidates): 100%
Processing candidates (89 candidates): 100%
Checking integrity... done (0 conflicting)
Your packages are up to date.
root@Lidarr:/usr/ports/lang/mono # 
```
Now, start Lidarr again:
```bash
root@Lidarr:/usr/ports/lang/mono # service lidarr start
Starting lidarr.
root@Lidarr:/usr/ports/lang/mono # 
```
#### make package
Create a tarball you can use on other systems, by issuing `make package`:
```bash
root@Lidarr:/usr/ports/lang/mono # make package
```

The package should now be in `work/pkg/`. 
```bash
root@Lidarr:/usr/ports/lang/mono/work/pkg # ls -alh
total 71356
drwxr-xr-x  2 root  wheel     3B Dec 22 23:09 .
drwxr-xr-x  8 root  wheel    23B Dec 22 23:11 ..
-rw-r--r--  1 root  wheel    70M Dec 22 23:11 mono-5.20.1.34_2.txz
root@Lidarr:/usr/ports/lang/mono/work/pkg # 
```

This file you can now copy to your other jails.
Then to install it, just do `pkg add -f /tmp/mono.mono-5.20.1.34_2.txz` in your other iocage jail. 



## Authors
Mr. Johnson
## Acknowledgments
* [https://github.com/Lidarr/Lidarr/wiki/Installation#linux](https://github.com/Lidarr/Lidarr/wiki/Installation#linux)
* [https://www.ixsystems.com/community/threads/how-to-giving-plugins-write-permissions-to-your-data.27273/](https://www.ixsystems.com/community/threads/how-to-giving-plugins-write-permissions-to-your-data.27273/)
* [https://github.com/lidarr/Lidarr/wiki/Installation-(FreeBSD-FreeNAS)](https://github.com/lidarr/Lidarr/wiki/Installation-(FreeBSD-FreeNAS))
* [https://unix.stackexchange.com/questions/483990/change-between-quarterly-and-latest-package-set-used-by-pkg-tool-in-freebs](https://unix.stackexchange.com/questions/483990/change-between-quarterly-and-latest-package-set-used-by-pkg-tool-in-freebs)
* [https://forums.freebsd.org/threads/update-fails-with-error-70-because-of-problematic-file-xorgproto-2018-4-with-xproto-7-0-31.67114/](https://forums.freebsd.org/threads/update-fails-with-error-70-because-of-problematic-file-xorgproto-2018-4-with-xproto-7-0-31.67114/)
* [https://forums.freebsd.org/threads/creating-binary-packages-without-installing.55238/](https://forums.freebsd.org/threads/creating-binary-packages-without-installing.55238/)

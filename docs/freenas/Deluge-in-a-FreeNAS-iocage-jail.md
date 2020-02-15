---
layout: default
title: How to install Deluge in a FreeNAS iocage jail
parent: FreeNAS
nav_order: 3
---

# How to install Deluge in a FreeNAS iocage jail
{: .no_toc }
I have Deluge 1.3.5 installed in an old warden jail on FreeNAS. This is how I replaced the warden jail with a new iocage jail with Deluge 2.0.3 on FreeNAS. This is the information that had me started: [https://forum.deluge-torrent.org/viewtopic.php?t=55437](https://forum.deluge-torrent.org/viewtopic.php?t=55437).

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

Select and START Deluge.

## Preconfiguration
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

### Install git
```bash
ross@Deluge:~ # pkg install git
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
The following 25 package(s) will be affected (of 0 checked):

New packages to be INSTALLED:
	git: 2.23.0
	p5-CGI: 4.44
	p5-HTML-Parser: 3.72
	p5-HTML-Tagset: 3.20_1
	perl5: 5.30.0
	expat: 2.2.8
	p5-IO-Socket-SSL: 2.066
	p5-Mozilla-CA: 20180117
	p5-Net-SSLeay: 1.85
	p5-IO-Socket-INET6: 2.72_1
	p5-Socket6: 0.29
	p5-Authen-SASL: 2.16_1
	p5-GSSAPI: 0.28_1
	p5-Digest-HMAC: 1.03_1
	python36: 3.6.9_1
	readline: 8.0.0
	indexinfo: 0.3.1
	libffi: 3.2.1_3
	gettext-runtime: 0.20.1
	p5-Error: 0.17028
	curl: 7.66.0
	libnghttp2: 1.39.2
	ca_root_nss: 3.47.1
	pcre: 8.43_2
	cvsps: 2.1_2

Number of packages to be installed: 25

The process will require 213 MiB more space.
39 MiB to be downloaded.

Proceed with this action? [y/N]: y
```
Change directory to `/home/deluge/`.

Go to [https://github.com/deluge-torrent/deluge](https://github.com/deluge-torrent/deluge]) and copy the clone link; [https://github.com/deluge-torrent/deluge.git](https://github.com/deluge-torrent/deluge.git]). 

then
## Download deluge
```bash
ross@Deluge:/home/deluge # git clone https://github.com/deluge-torrent/deluge.git
Cloning into 'deluge'...
remote: Enumerating objects: 11, done.
remote: Counting objects: 100% (11/11), done.
remote: Compressing objects: 100% (8/8), done.
remote: Total 96736 (delta 5), reused 6 (delta 3), pack-reused 96725
Receiving objects: 100% (96736/96736), 60.37 MiB | 17.06 MiB/s, done.
Resolving deltas: 100% (69232/69232), done.
```

Check out the dependencies for deluge which is stated in `DEPENDS.md`:
```bash
root@Deluge:/home/deluge/deluge # more DEPENDS.md
```

## Install dependencies
```bash
root@Deluge:/home/deluge/deluge # pkg install nano intltool closure-compiler py36-twisted  py36-OpenSSL py36-rencode py36-rencode py36-xdg  xdg-utils py36-six py36-zope.interface py36-chardet py36-setproctitle py36-pillow py36-dbus py36-distro py36-libtorrent-rasterbar py36-GeoIP2 py36-mako libnotify
```

## Build deluge
```bash
ross@Deluge:/home/deluge # cd deluge/
ross@Deluge:/home/deluge/deluge # python3.6 setup.py build
```

## Install deluge
```bash
ross@Deluge:/home/deluge # cd deluge/
ross@Deluge:/home/deluge/deluge # python3.6 setup.py install
```

## Startup script deluge_web
### /etc/rc.conf
Edit rc.conf and add the following lines, change `deluge_web_user` to the user you would like `deluge_web` to run as on your system (I am using `deluge`):
```bash
ross@Deluge:/home/deluge/deluge # nano /etc/rc.conf
# Enable deluge web service
deluge_web_enable="YES"
deluge_web_user="deluge"
```

### /usr/local/etc/rc.d/deluge_web
```bash
ross@Deluge:/home/deluge/deluge # cd /usr/local/etc/rc.d/
ross@Deluge:/usr/local/etc/rc.d # nano deluge_web
#!/bin/sh

# $FreeBSD: head/net-p2p/deluge-cli/files/deluge_web.in 418935 2016-07-22 20:49:47Z rm $
#
# PROVIDE: deluge_web
# REQUIRE: LOGIN
# KEYWORD: shutdown
#
# Add the following lines to /etc/rc.conf.local or /etc/rc.conf
# to enable this service:
#
# MANDATORY:
#
# deluge_web_enable (bool):     Set to NO by default.
#                               Set it to YES to enable deluge_web.
#
# deluge_web_user (str):        The UNPRIVILEGED user to run as
#
# OPTIONAL:
#
# deluge_web_flags (str):       Set as needed
#                               See deluge-web(1) for more information
#
# deluge_web_confdir (path):    Set to /home/$deluge_web_user/.config/deluge
#                               by default
#
# deluge_web_loglevel (str):    Set to "error" by default
#
# deluge_web_logfile (path):    Set to /var/tmp/deluge_web.log by default

. /etc/rc.subr

name="deluge_web"
rcvar=${name}_enable

command=/usr/local/bin/deluge-web
command_interpreter=/usr/local/bin/python3.6

start_precmd=${name}_prestart
stop_postcmd=${name}_poststop

deluge_web_prestart()
{
        if [ "$deluge_web_user" = 'asjklasdfjklasdf' ]; then
                err 1 "You must set deluge_web_user to a real, unprivileged user"
        fi

        if [ ! -d "/var/run/${name}" ]; then
                if [ -e "/var/run/${name}" ]; then
                        unlink /var/run/${name}
                fi
                mkdir -p /var/run/${name}
        fi

        if [ ! -d "/home/${deluge_web_user}/.python-eggs" ]; then
                mkdir -p /home/${deluge_web_user}/.python-eggs
        fi

        chmod 0755 /var/run/${name}
        chown -R $deluge_web_user /var/run/${name}
        chown -R $deluge_web_user /home/${deluge_web_user}/.python-eggs
        export PYTHON_EGG_CACHE="/home/${deluge_web_user}/.python-eggs"
}

deluge_web_poststop()
{
        [ -e "$deluge_web_logfile" -a ! -s "$deluge_web_logfile" ] &&
                unlink $deluge_web_logfile
}

load_rc_config $name

: ${deluge_web_enable:="NO"}
: ${deluge_web_user:="asjklasdfjklasdf"}
: ${deluge_web_confdir:="/home/${deluge_web_user}/.config/deluge"}
: ${deluge_web_loglevel:="error"}
: ${deluge_web_logfile:="/var/tmp/${name}.log"}

required_dirs="$deluge_web_confdir"
command_args="-f -c $required_dirs -L $deluge_web_loglevel -l $deluge_web_logfile"

run_rc_command "$1"
```

Change chmod:
```bash
ross@Deluge:/usr/local/etc/rc.d # chmod 555 deluge_web
```
### Data-dir
Create directories for deluge services to startup
```bash
ross@Deluge:~ # cd /home/deluge/
ross@Deluge:/home/deluge # mkdir .config
ross@Deluge:/home/deluge # cd .config
ross@Deluge:/home/deluge/.config # mkdir deluge 
ross@Deluge:/home/deluge/.config # chown -R deluge:deluge /home/deluge/.config
```

## Start deluge_web 
Start our deluge_web service:
```bash
ross@Deluge:/home/deluge # /usr/local/etc/rc.d/deluge_web start
Starting deluge_web.
```

### 8112
Navigate to localhost:8112 to see if the deluge webs service is up and running. From the web service, you can start `deluged` (Deluge daemon) through the Connection Manager. 

## Startup script deluged
If you do not want to have deluge start the daemon automatically, but started from the web service, skip this entire step. 
### /etc/rc.conf
Add the following line to `/etc/rc.conf` to enable deluged at startup
  `deluged_enable="YES"`
```bash
ross@Deluge:/home/deluge/deluge # nano /etc/rc.conf
# Enable deluged
deluged_enable="YES"
```

#### /usr/local/etc/rc.d/deluged
```bash
ross@Deluge:/home/deluge/deluge # cd /usr/local/etc/rc.d/
ross@Deluge:/usr/local/etc/rc.d # nano deluge_web
#!/bin/sh
#
#  deluged RCng startup script
#  created by: R.S.A. aka .faust 
#            mail: rsa dot aka dot f at gmail dot com
# 
 
# PROVIDE: deluged
# REQUIRE: NETWORKING SERVERS DAEMON ldconfig resolv
# BEFORE: LOGIN
# KEYWORD: shutdown
 
# Add the following line to /etc/rc.conf.local or /etc/rc.conf to enable deluged at startup
#       deluged_enable="YES"
#
# cfg_dir (str):        Specify the full path to directory with deluged config files
# log (str):            Specify the full path to the LOG file
# loglevel (str):       Set loglevel (Available: none, info, warning, error, critical, debug)
# pidfile (str):        Specify the full path to the PID file
# deluged_user (str):   Set to user running deluged
#
#  Warning! Rights to folders and files must be "rwx" for the user under which deluged is run
 
. /etc/rc.subr
 
name="deluged"
rcvar=`set_rcvar`
 
load_rc_config $name
deluged_enable=${deluged_enable:=NO}

cfg_dir="/home/deluge/.config/deluge/"
log="${cfg_dir}${name}.log"
loglevel="error"
pidfile="${cfg_dir}${name}.pid"
deluged_user="deluge"

required_dirs=${cfg_dir}

command_interpreter="/usr/local/bin/python"
command="/usr/local/bin/${name}"
start_cmd="${name}_start"

deluged_start()
{
if [ ! -f "${pidfile}" ]; then
    su -m ${deluged_user} -c "/usr/local/bin/${name} -c ${cfg_dir} -L ${loglevel} -l ${log} -P ${pidfile}"
    echo "Starting ${name}."
else
    GETPROCESSPID=`/bin/ps -auxw | /usr/bin/awk '/deluged/ && !/awk/ && !/sh/ {print $2}'`
    PIDFROMFILE=`cat ${pidfile}`
    if [ "$GETPROCESSPID" = "$PIDFROMFILE" ]; then
        echo "${name} already running with PID: ${PIDFROMFILE} ?"  
        echo "Remove ${pidfile} manually if needed."
    else
        rm -f ${pidfile}
        su -m ${deluged_user} -c "/usr/local/bin/${name} -c ${cfg_dir} -l ${log} -P ${pidfile}"
        echo "Starting ${name}."
    fi
fi
}
run_rc_command "$1"
```

```bash
root@Deluge:/usr/local/etc/rc.d # chmod +x deluged 
```


## Configuration of Deluge
### Downloads
Download to: /mnt/Torrents/Transmission
Move completed to: /mnt/Torrents/Finished
Copy of .torrent files to: /mnt/Torrents/Old
Autoadd .torrent files from: /mnt/Torrents/Watch
Allocation:
Use Compact

### Encryption
Inbound: Forced
Outbound: Forced
Level: Either
* Encrypt entire stream

### Bandwitdth
Maximum Connections: -1
Maximum Upload Slots: 15
Maximum Download Speed (KiB/s): -1
Maximum Upload Speed (KiB/s): -1
Maximum Half-Open Connections: 50
Maximum Connection Attempts per Second: 20

* Ignore limits on local network
* Rate limit IP overhead

Per Torrent Bandwidth Usage
Maximum Connections: -1
Maximum Upload Slots: -1
Maximum Download Speed (KiB/s): -1
Maximum Upload Speed (KiB/s): -1

### Interface
* Allow the use of multiple filters at once

## Other
GeoIP.dat is no longer maintained by MaxMind, but there is a guy converting the new GeoIP version 2 files `*.mmdb`  to `*.dat` which you can get here; [https://www.miyuru.lk/geoiplegacy](https://www.miyuru.lk/geoiplegacy).

Anyways - even if you have the "old" `GeoIP.dat` files, we currently just have to wait for the deluge team to use GeoIP2 instead of the old legacy GeoIP python extension, as it is removed from freshports [https://www.freshports.org/net/py-GeoIP/](https://www.freshports.org/net/py-GeoIP/) -- or maybe you could download and install the old one? [https://github.com/maxmind/geoip-api-python](https://github.com/maxmind/geoip-api-python).

### Download old GeoIP Legacy C API
[https://github.com/maxmind/geoip-api-c](https://github.com/maxmind/geoip-api-c)
```bash
root@Deluge:/home/deluge # git clone https://github.com/maxmind/geoip-api-c.git
Cloning into 'geoip-api-c'...
remote: Enumerating objects: 3649, done.
remote: Total 3649 (delta 0), reused 0 (delta 0), pack-reused 3649
Receiving objects: 100% (3649/3649), 1.75 MiB | 2.91 MiB/s, done.
Resolving deltas: 100% (2402/2402), done.
root@Deluge:/home/deluge # 
```
Hm. Do not understand how I could get this installed. `./configure` which is suggested, is probably pointing to the `configure.ac` file. [https://forums.freebsd.org/threads/configure.32843/](https://forums.freebsd.org/threads/configure.32843/). 

Blaergh. 

### Download old Legacy GeoIP
[https://github.com/maxmind/geoip-api-python](https://github.com/maxmind/geoip-api-python)
```bash
ross@Deluge:/home/deluge # git clone https://github.com/maxmind/geoip-api-python.git
Cloning into 'geoip-api-python'...
remote: Enumerating objects: 441, done.
remote: Total 441 (delta 0), reused 0 (delta 0), pack-reused 441
Receiving objects: 100% (441/441), 103.70 KiB | 643.00 KiB/s, done.
Resolving deltas: 100% (251/251), done.
root@Deluge:/home/deluge # 
```
```bash
root@Deluge:/home/deluge/geoip-api-python # python3.6 setup.py build
/usr/local/lib/python3.6/distutils/dist.py:261: UserWarning: Unknown distribution option: 'bugtrack_url'
  warnings.warn(msg)
running build
running build_ext
building 'GeoIP' extension
creating build
creating build/temp.freebsd-11.2-STABLE-amd64-3.6
cc -Wno-unused-result -Wsign-compare -Wunreachable-code -DNDEBUG -O2 -pipe -fstack-protector-strong -fno-strict-aliasing -fPIC -I/usr/local/include/python3.6m -c py_GeoIP.c -o build/temp.freebsd-11.2-STABLE-amd64-3.6/py_GeoIP.o
py_GeoIP.c:23:10: fatal error: 'GeoIP.h' file not found
#include <GeoIP.h>
         ^~~~~~~~~
1 error generated.
error: command 'cc' failed with exit status 1
root@Deluge:/home/deluge/geoip-api-python # 
```

GeoIP Database
* Location: /usr/local/share/GeoIP/GeoIP.dat
*Location: /usr/home/deluge/GeoIP/GeoIP.dat

Yah, blaergh.
## Daemon 
Daemon port: 58846
Connections
* Allow Remote Connections
Other

## Queue
* Queue new torrents on top
Active Torrents
Total: 299
Downloading: 250
Seeding: 50
* Ignore slow torrents

Seeding Rotation
Share Ratio: 0
Time Ratio: 0
Time (m): 0

Share Ratio Reached
* Share Ratio: 0
 * Remove torrent
* Stop seeding when share ratio reaches 0.5
 * Remove torrent when share ratio is reached

## Proxy

## Plugins
* AutoAdd
* Blocklist
* Label

Restart the deluge jail.

## Labels
### lidarr
Queue
v Apply queue settings:
 v Auto Managed
  v Stop seed at ratio: 0
  v Remove at ratio

Folders
v Apply folder settings:
 v Move completed to: 
  /mnt/Torrents/Finished/lidarr_finished

### mylar
Queue
v Apply queue settings:
 v Auto Managed
  v Stop seed at ratio: 0
  v Remove at ratio

Folders
v Apply folder settings:
 v Move completed to: 
  /mnt/Mylar_dump/DUMP/

### tv-sonarr
Queue
v Apply queue settings:
 v Auto Managed
  v Stop seed at ratio: 0

### radarr
Queue
v Apply queue settings:
 v Auto Managed
  v Stop seed at ratio: 0

### lazylibrarian
Queue
v Apply queue settings:
 v Auto Managed
  v Stop seed at ratio: 0
  v Remove at ratio

Folders
v Apply folder settings:
 v Move completed to: 
  /mnt/Torrents/Finished/LazyLibrarian

   
### Install YaRSS
#### First install Deluge 
This I did on my Mint linux edition.

The ​Deluge PPA contains the latest Deluge releases for Ubuntu.
```bash
sudo add-apt-repository ppa:deluge-team/stable
sudo apt-get update
sudo apt-get install deluge
```

Start deluge by issuing `deluge`. Change to Thin Client in Preferences. Quit deluge. 

Then run deluge with the following parameters:
```bash
deluge "connect ip-adress:port username:password"
```

See "Misc - Create users for daemond" for further information on how to create users. 

Download [https://bitbucket.org/bendikro/deluge-yarss-plugin/downloads/](https://bitbucket.org/bendikro/deluge-yarss-plugin/downloads/): YaRSS2-2.1.4-py3.6.egg

Go to Preferences, Plugins and press Install within the Deluge client. If nothing is happening, verify that you are using a correct python version which is corresponding with the version listed in the *.egg.
#### Add RSS Feeds
Go to Preferences, select YaRSS2 - and then select RSS Feeds.

Click Add Feed:
RSS Feed Name: 
RSS Feed URL:
Update Interval (min): 120 v Run on startup
Obey TTL: v Use TTL value from RSS Feed (recommended)
Cookies:
Magnet link: v Prefer magnet link over torrent

#### Add Subscriptions
Go to Preferences, select YaRSS2 - and then select Subscriptions.

Click Add Subscription
Subscription name:
RSS Feed:
Filter include (regex)
Filter exclude (regex)

Options
Label: 

## AutoAdd
Go to Preferences, select AutoAdd

/mnt/Torrents/Watch

## Misc
### Create users for daemond
You can create users in the `auth` file.
```bash
ross@Deluge:/home/deluge # cd /home/deluge/.config/deluge/
ross@Deluge:/home/deluge/.config/deluge # nano auth
```

### Old jail
deluge:*:922:922:Deluge BitTorrent Client:/home/deluge:/sbin/nologin
ross@Deluge:/ # groups deluge
deluge mylar_dump

groups deluge
mylar_dump:*:1049:deluge

ross@Deluge:/mnt # ls -alh
total 578
drwxr-xr-x     5 ross  wheel          5B Oct  4  2018 .
drwxr-xr-x    18 ross  wheel         22B Mar 28  2018 ..
drwxrwx---+ 1209 1000  1002         1.2K Dec 16 16:25 Movies
drwxrwx---    12 ross  mylar_dump    15B Dec  8 00:20 Mylar_dump
drwxrwx---+    7 1000  deluge         9B May  7  2018 Torrents
ross@Deluge:/mnt # 


### Groups
Create a group called `mylar_dump` with GID = 1049 in the iocage jail.
```bash
ross@Deluge:~ # pw groupadd mylar_dump -g 1049
ross@Deluge:~ # groups deluge
deluge
ross@Deluge:~ # pw usermod deluge -G deluge,mylar_dump
ross@Deluge:~ # groups deluge
deluge mylar_dump
ross@Deluge:~ # 
```

### Mount points
I have to mount Movies, Mylar_dump and Torrents.
And create a group called mylar_dump with GID=1049.

And then turn off all torrents on the old warden jail, shut down the warden jail - mount the mount points, and then edit the IP of the new iocage jail. And then test.

## Execute
 Torrent Complete /home/deluge/echo_script.sh
 ```bash
 ross@Deluge:/home/deluge # more echo_script.sh 
#!/bin/bash
torrentid=$1
torrentname=$2
torrentpath=$3
echo "Torrent Details: " "$torrentname" "$torrentpath" "$torrentid"  >> /home/deluge/echo_script.log
```

## Fault finding
Set error logging to either
* none
* critical
* error
* warning
* info
* debug

Note: `debug` is very verbose and with a lot of torrents log files will be MB's in size.

You change the log levels your `/usr/local/etc/rc.d/deluge_web` and or `/usr/local/etc/rc.d/deluged` file.

Logging for `deluged`:
```bash
root@Deluge:~ # tail -f /home/deluge/.config/deluge/deluged.log
```

Logging for `deluge_web`:
```bash
root@Deluge:~ # tail -f /var/tmp/deluge_web.log
```

## Authors
Mr. Johnson

## Acknowledgments
* [https://computingforgeeks.com/how-to-install-pip-python-package-manager-on-freebsd-12/](https://computingforgeeks.com/how-to-install-pip-python-package-manager-on-freebsd-12/)
* [https://blog.tkrn.io/deluge-in-a-freenas-11-iocage-jail-easy-tutorial/](https://blog.tkrn.io/deluge-in-a-freenas-11-iocage-jail-easy-tutorial/)
* [https://www.freshports.org/net-p2p/deluge/](https://www.freshports.org/net-p2p/deluge/)
* [https://github.com/deluge-torrent/deluge/blob/develop/DEPENDS.md](https://github.com/deluge-torrent/deluge/blob/develop/DEPENDS.md)
* [https://computingforgeeks.com/how-to-install-pip-python-package-manager-on-freebsd-12/](https://computingforgeeks.com/how-to-install-pip-python-package-manager-on-freebsd-12/)
* [https://forum.deluge-torrent.org/viewtopic.php?f=7&t=55437&p=230328&hilit=freebsd#p230328](https://forum.deluge-torrent.org/viewtopic.php?f=7&t=55437&p=230328&hilit=freebsd#p230328)
* [https://stackoverflow.com/questions/21050456/how-to-convert-a-maxmind-mmdb-to-dat/21150601#21150601](https://stackoverflow.com/questions/21050456/how-to-convert-a-maxmind-mmdb-to-dat/21150601#21150601)
* [https://www.miyuru.lk/geoiplegacy](https://www.miyuru.lk/geoiplegacy)
* [https://github.com/sherpya/geolite2legacy](https://github.com/sherpya/geolite2legacy)
* [https://github.com/transmission-remote-gui/transgui/issues/1104](https://github.com/transmission-remote-gui/transgui/issues/1104)
* [https://github.com/transmission-remote-gui/transgui/issues/1195](https://github.com/transmission-remote-gui/transgui/issues/1195)
* [https://dev.deluge-torrent.org/wiki/UserGuide/Service/FreeBSD](https://dev.deluge-torrent.org/wiki/UserGuide/Service/FreeBSD)
* [https://forum.deluge-torrent.org/viewtopic.php?t=55382](https://forum.deluge-torrent.org/viewtopic.php?t=55382)
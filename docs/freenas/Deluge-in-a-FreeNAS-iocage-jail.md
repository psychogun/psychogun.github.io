---
layout: default
title: How to install Deluge in a FreeNAS iocage jail
parent: FreeNAS
nav_order: 3
---

# How to install Deluge in a FreeNAS iocage jail
{: .no_toc }
How to install Deluge 2.0.3 in a FreeNAS iocage jail.

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

Go to https://github.com/deluge-torrent/deluge and copy the clone link; https://github.com/deluge-torrent/deluge.git. 

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
root@Deluge:/home/deluge/deluge # pkg install nano intltool closure-compiler py36-twisted  py36-OpenSSL py36-rencode py36-rencode py36-xdg  xdg-utils py36-six py36-zope.interface py36-chardet  py36-setproctitle py36-pillow py36-dbus py36-distro py36-libtorrent-rasterbar py36-GeoIP2 py36-mako libnotify
```

pyOpenSSL
rencode
py36-xdg-utils
py36-xdg-utils
setproctitle

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

## Startup script
### /etc/rc.conf
Edit rc.conf and add the following lines, change `deluge_web_user` to the user you would like deluge_web to run as on your system (I am using `deluge`):
```bash
ross@Deluge:/home/deluge/deluge # nano /etc/rc.conf
# Enable deluge 
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
Navigate to localhost:8112 to see if it is up and running by starting the Daemon through the Connection Manager.

## Configuration of Deluge


## Authors
Mr. Johnson

## Acknowledgments
* [https://computingforgeeks.com/how-to-install-pip-python-package-manager-on-freebsd-12/](https://computingforgeeks.com/how-to-install-pip-python-package-manager-on-freebsd-12/)
https://blog.tkrn.io/deluge-in-a-freenas-11-iocage-jail-easy-tutorial/
* [https://www.freshports.org/net-p2p/deluge/](https://www.freshports.org/net-p2p/deluge/)
* [https://github.com/deluge-torrent/deluge/blob/develop/DEPENDS.md](https://github.com/deluge-torrent/deluge/blob/develop/DEPENDS.md)
* [https://computingforgeeks.com/how-to-install-pip-python-package-manager-on-freebsd-12/](https://computingforgeeks.com/how-to-install-pip-python-package-manager-on-freebsd-12/)
* [https://forum.deluge-torrent.org/viewtopic.php?f=7&t=55437&p=230328&hilit=freebsd#p230328](https://forum.deluge-torrent.org/viewtopic.php?f=7&t=55437&p=230328&hilit=freebsd#p230328)
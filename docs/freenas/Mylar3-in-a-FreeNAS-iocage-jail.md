---
layout: default
title: How to install Mylar3 in a FreeNAS iocage jail
parent: FreeNAS
nav_order: 10
---

# How to install Mylar3 in a FreeNAS iocage jail
{: .no_toc }
From 

## Table of contents
{: .no_toc .text-delta }
1. TOC
{:toc}
---

# Prerequisites
* FreeNAS Version: FreeNAS-11.3-RELEASE
* Iocage jail version: 11.3-RELEASE

## Getting started
Go to your FreeNAS. Select Jails > and click Add.
* Name: Mylar3
* Jail Type: Default (Clone Jail)
* Release*: 11.3-RELEASE
Click NEXT.

* [v] DHCP Autoconfigure IPv4
* [v] VNET
Click NEXT and confirm the settings with clicking SUBMIT. 

Start your jail by selecting it and then click START.

### update && upgrade
SSH in to your FreeNAS and log in to your jail with the command `iocage console Mylar3`:

```bash
root@freenas:~ # iocage console Mylar3
root@Mylar3:~ # pkg update
root@Mylar3:~ # pkg upgrade
```

### Python?
Which version of python do you currently have installed on your system?
```bash
root@Mylar3:/usr/local # python3.7 -V
Python 3.7.6
```

## Install the necessary requirements
There are some requirements for running `mylar3`. These can be found here [https://github.com/mylar3/mylar3/blob/master/requirements.txt](https://github.com/mylar3/mylar3/blob/master/requirements.txt).

I think I've got them all. Let us install them (this process will require 540 MiB more space, 112 MiB (116 items) to be downloaded):
```bash
root@Mylar3:~ # pkg install screen nano git py37-sqlite3 py37-apscheduler py37-cherrypy py37-requests py37-beautifulsoup py37-pip py37-feedparser py37-portend py37-mako py37-six unrar py37-natsort py37-configparser py37-cheroot py37-cloudflare-scrape py37-pyinstaller py37-pillow py37-pytz py37-simplejson py37-tzlocal py37-urllib3
```

I was unable to locate `unrar-cffi`, but let us install it with pip: 

??

==0.1.0a5

## Add a service user
Add a user which will act as a "service" user to start `mylar3`. This user is called `mylar` with `uid=8675309`, has `/nonexistent` home directory and sets the user's login shell to `/usr/sbin/nologin` which denies this user interactive login- and a comment, `-c`.
```bash
root@Mylar3:/usr/local/etc/rc.d # pw adduser mylar -u 8675309 -d /nonexistent -s /usr/sbin/nologin -c "Mylar service user for mylar3"
```

## Install mylar3
We'll use `git clone` and this will create a folder called `mylar3` in the working current directory. Let us do this under the `/usr/local/` folder:
```bash
root@Mylar3:~ # cd /usr/local/
root@Mylar3:/usr/local # git clone https://github.com/mylar3/mylar3.git
```

## Configuration of mylar3
### chown
Make our service user `mylar` the owner of the newly created `mylar3` folder, and do this recursively:
```bash
root@Mylar3:/usr/local # chown -R mylar mylar3/
```

### Manual start
If you would like to manually start `mylar3` with the user `mylar` and all the default settings, I suggest doing this:
```bash
root@Mylar3:/usr/local/mylar3 # screen
root@Mylar3:/usr/local/mylar3 # su -m mylar -c '/usr/local/bin/python3.7 /usr/local/mylar3/Mylar.py'
```
Use CTRL + A + D to detach from screen and `mylar` will be running in the "background".

You will now be able to access `mylar3`, if you open http://ip-adress:8090/home.

To reattach the `screen` session, write `screen -r`. 

### Automatic start
To make `mylar3` start at boot of the jail, do the following:

#### python symlink
```bash
root@Mylar3:/usr/local/ # cd bin/
root@Mylar3:/usr/local/bin # ls -alh | grep python
-r-xr-xr-x    2 root  wheel   5.4K Jan 30 02:18 python3.7
lrwxr-xr-x    1 root  wheel    17B Jan 30 02:19 python3.7-config -> python3.7m-config
-r-xr-xr-x    2 root  wheel   5.4K Jan 30 02:18 python3.7m
-r-xr-xr-x    1 root  wheel   2.9K Jan 30 02:19 python3.7m-config
```

Make a python symlink from `python3.7` to `python`. This is necessary in the rc script we'll be using next (or just edit the script with the correct path to a python executable):

```bash
root@Mylar3:/usr/local/etc/rc.d # ln -s /usr/local/bin/python3.7 /usr/local/bin/python
```
Voilà:
```bash
root@Mylar3:/usr/local/bin # ls -alh | grep python
lrwxr-xr-x    1 root  wheel    24B Feb 29 23:23 python -> /usr/local/bin/python3.7
-r-xr-xr-x    2 root  wheel   5.4K Jan 30 02:18 python3.7
lrwxr-xr-x    1 root  wheel    17B Jan 30 02:19 python3.7-config -> python3.7m-config
-r-xr-xr-x    2 root  wheel   5.4K Jan 30 02:18 python3.7m
-r-xr-xr-x    1 root  wheel   2.9K Jan 30 02:19 python3.7m-config
```

#### rc.d
Create a file called `mylar3` in `/usr/local/etc/rc.d/`:
```bash
root@Mylar3:~ # nano /usr/local/etc/rc.d/mylar3

#!/bin/sh
#
# PROVIDE: mylar3
# REQUIRE: DAEMON
# BEFORE:  LOGIN
# KEYWORD: shutdown
#
# Add the following line to /etc/rc.conf.local or /etc/rc.conf
# to enable this service:
#           mylar3_enable="YES"
# Start it with:
#           service mylar3 start
#
# Stop it with:
#           service mylar3 stop
#
#
#
#
#
#
# mylar3_user:  The user account Mylar daemon runs as what
#           you want it to be. It uses 'mylar' user by
#           default. Do not sets it as empty or it will run
#           as root.
# mylar3_dir:   Directory where Mylar lives.
#           Default: /usr/local/mylar3
# mylar3_pid:  The name of the pidfile to create.
#           Default is mylar.pid in mylar_dir.
# mylar3_conf: The name of the config file you would like to launch with mylar3.

. /etc/rc.subr

name="mylar3"
rcvar="${name}_enable"

load_rc_config ${name}
: "${mylar3_enable:="NO"}"
: "${mylar3_user:="mylar"}"
: "${mylar3_dir:="/usr/local/mylar3"}"
: "${mylar3_conf:="/usr/local/mylar3/config.ini"}"

pidfile="/var/run/mylar3/mylar3.pid"

command="/usr/local/bin/python"
command_args="${mylar3_dir}/Mylar.py --daemon --nolaunch --pidfile $pidfile --config $mylar3_conf"


start_precmd="${name}_start_precmd"
mylar3_start_precmd() {
        if [ $($ID -u) != 0 ]; then
                err 1 "Must be root."
        fi

        if [ ! -d /var/run/$name ]; then
                install -do $mylar3_user /var/run/$name
        fi
}

load_rc_config ${name}
run_rc_command "$1"
```

Make the newly created file executable:
```bash
root@Mylar3:/usr/local/etc/rc.d # chmod +x /usr/local/etc/rc.d/mylar3 
```
Add the following line to /etc/rc.conf.local or /etc/rc.conf to enable this service:
```bash
root@Mylar3:/usr/local/mylar3 # nano /etc/rc.conf

(...)
# Enable Mylar
mylar3_enable="YES"
```

Start `mylar3`:
```bash
root@Mylar3:/usr/local/etc/rc.d # service mylar3 start
Starting mylar3.
log language set to en_US
Initializing startup sequence....
Unable to make proper backup of config file in /usr/local/mylar3/config.ini.backup
Attempting to update configuration..
Updating Configuration from 0 to 10
Checking for existing torznab configuration...
No existing torznab configuration found. Just removing config references at this point..
Successfully removed outdated config entries.
Configuration upgraded to version 10
29-Feb-2020 23:26:24 - ERROR :: mylar.configure.955 : MainThread : No User Comicvine API key specified. I will not work very well due to api limits - http://api.comicvine.com/ and get your own free key.
29-Feb-2020 23:26:24 - INFO :: mylar.configure.1009 : MainThread : [COMICTAGGER] Version detected: 1.3.1
29-Feb-2020 23:26:24 - INFO :: mylar.initialize.186 : MainThread : Checking to see if the database has all tables....
29-Feb-2020 23:26:24 - WARNING :: mylar.dbcheck.676 : MainThread : Unable to update readinglist table to new storyarc table format.
29-Feb-2020 23:26:25 - INFO :: mylar.dbcheck.1318 : MainThread : Ensuring DB integrity - Removing all Erroneous Comics (ie. named None)
29-Feb-2020 23:26:25 - INFO :: mylar.dbcheck.1320 : MainThread : Correcting Null entries that make the main page break on startup.
29-Feb-2020 23:26:25 - INFO :: mylar.dbcheck.1333 : MainThread : Updating db to include some important changes.
29-Feb-2020 23:26:25 - INFO :: mylar.upgrade_dynamic.1301 : MainThread : Finished updating 0 / 0 entries within the db.
29-Feb-2020 23:26:25 - INFO :: mylar.initialize.200 : MainThread : Successfully discovered local IP and locking it in as : 192.168.5.115
29-Feb-2020 23:26:25 - INFO :: mylar.initialize.262 : MainThread : Remapping the sorting to allow for new additions.
29-Feb-2020 23:26:25 - INFO :: mylar.ComicSort.761 : MainThread : Sucessfully ordered -1 series in your watchlist.
29-Feb-2020 23:26:25 - INFO :: mylar.validateAndCreateDirectory.1693 : MainThread : [DIRECTORY-CHECK] Found comic directory: /usr/local/mylar3
root@Mylar3:/usr/local/etc/rc.d # 
```


## Fault finding
### Status
Check if `mylar3` is running (as a service):

```bash
root@Mylar3:~ # service mylar3 status
mylar3 is running as pid 10075.
```

Find the PID of `mylar3` with `ps` (here the command is used with pipe, `|`, and `grep` so you'll be able to filter the output):
```bash
root@Mylar3:~ # ps -aux | grep mylar3
mylar 10075  0.0  0.1 98004 61108  -  SJ   22:36   0:00.13 /usr/local/bin/python /usr/local/mylar3/Mylar.py --daemon --nolaunch --pidfile /var/run/mylar3/mylar3.pid --config /usr/local/mylar3/config.ini (python3.7)
```

### Change branch
If you would like to change `mylar3` from the `master` branch to `python3-dev` (which get updates more frequently), do this:

Stop `mylar3` from running with `service mylar3 stop` (or use `screen -r` to reattach to your screen session and then press CTRL + C on your keyboard). 

Change from `master` branch to `python3-dev`:
```bash
root@Mylar3:/usr/local/ # cd mylar3/
root@Mylar3:/usr/local/mylar3 # git checkout python3-dev
Branch 'python3-dev' set up to track remote branch 'python3-dev' from 'origin'.
Switched to a new branch 'python3-dev'
root@Mylar3:/usr/local/mylar3 # git status
On branch python3-dev
Your branch is up to date with 'origin/python3-dev'.

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	=2.0.8

nothing added to commit but untracked files present (use "git add" to track)
```
Edit your `config.ini` and change `git_branch = None` to `git_branch = python3-dev`:
```bash
root@Mylar3:/usr/local/mylar3 # nano config.ini

[Git]
(...)
git_branch = python3-dev
(...)
```
Save.


### Symbolic link pip
```bash
root@Mylar3:~ # 
root@Mylar3:/usr/local/mylar3 # ln -s /usr/local/bin/pip-3.7 /usr/local/bin/pip3
```

## Authors
Mr. Johnson

## Acknowledgments
* [https://computingforgeeks.com/how-to-install-pip-python-package-manager-on-freebsd-12/](https://computingforgeeks.com/how-to-install-pip-python-package-manager-on-freebsd-12/)
* [https://www.freebsd.org/doc/en_US.ISO8859-1/articles/rc-scripting/](https://www.reddit.com/r/nzbhydra/comments/8fqcyl/freebsd_install_guide/)
* [https://www.freebsd.org/cgi/man.cgi?pw(8)](https://www.freebsd.org/cgi/man.cgi?pw(8))


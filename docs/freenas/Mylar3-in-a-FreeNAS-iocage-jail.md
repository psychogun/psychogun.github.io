---
layout: default
title: How to install Mylar3 in a FreeNAS iocage jail
parent: FreeNAS
nav_order: 10
---

# How to install Mylar3 in a FreeNAS iocage jail
{: .no_toc }
This is how I installed mylar3 in an python virtual environment in a FreeNAS iocage jail. 

# Table of contents
{: .no_toc .text-delta }
1. TOC
{:toc}
---

# Prerequisites
* FreeNAS Version: FreeNAS-11.3-RELEASE
* Iocage jail version: 11.3-RELEASE

# Getting started
Go to your FreeNAS. Select Jails > and click Add.
* Name: Mylar3
* Jail Type: Default (Clone Jail)
* Release*: 11.3-RELEASE
Click NEXT.

* [v] DHCP Autoconfigure IPv4
* [v] VNET
Click NEXT and confirm the settings with clicking SUBMIT. 

## Add mount points
Add your mount points. 

Start your jail by selecting it and then click START.

## update && upgrade
SSH in to your FreeNAS and log in to your jail with the command `iocage console Mylar3`:

```bash
root@freenas:~ # iocage console Mylar3
root@Mylar3:~ # pkg update
root@Mylar3:~ # pkg upgrade
```

## Install some necessary dependencies
```bash
root@Mylar3:~ # pkg install gmake wget screen nano git sudo bash py37-virtualenv py37-pip lzlib openjpeg unrar py37-sqlite3
```

## Add a group
According to your ACLs on the mounted dataset(s), which groups have read/write access to the share? On my datasets, there is a group called `mylar` with a `GID=1048` and on another dataset there's a group called `mylar_dump` with a `GID=1049` that has read/write access.

These mounted datasets are for the Comic Location Path and the folder to monitor.

 Let's create these groups in the jail:
```bash
root@Mylar3:/etc/pkg # pw groupadd mylar -g 1048
root@Mylar3:/etc/pkg # pw groupadd mylar_dump -g 1049
```
## Add a service user
Add a user which will act as a service user to start `mylar3`. This user is called `mylar` with `uid=8675309`, has `/nonexistent` home directory and sets the user's login shell to `/usr/sbin/nologin` which denies this user interactive login- and a comment is also provided to this user, `-c`.
```bash
root@Mylar3:/etc/pkg # pw adduser mylar -u 8675309 -d /nonexistent -s /usr/sbin/nologin -c "Mylar service user for mylar3"
```

Which groups does this user belong to?
```bash
root@Mylar3:/usr/local # id mylar
uid=8675309(mylar) gid=1048(mylar) groups=1048(mylar)
```

Add the user `mylar` to our `mylar_dump` group:
```bash
root@Mylar3:/usr/local # pw usermod mylar -G mylar_dump
root@Mylar3:/usr/local # id mylar
uid=920(mylar) gid=1048(mylar) groups=1048(mylar),1049(mylar_dump)
```

## Create a virtual environment
To create a python virtual environment, which will be identified as `(mylar)`, inside of `/usr/local/virtual` folder, do this:
```
root@Mylar3:/usr/local/virtual # python3.7 -m venv /usr/local/virtual/mylar
```

## Activate the virtual environment
```bash
root@Mylar3:/usr/local/virtual # cd /usr/local/virtual/mylar/bin
root@Mylar3:/usr/local/virtual/mylar/bin # bash
[root@Mylar3 /usr/local/virtual/mylar/bin]# source ./activate
(mylar) [root@Mylar3 /usr/local/virtual/mylar/bin]# 
```
# Requirements.txt
There are some requirements for running `mylar3`. These can be found here [https://github.com/mylar3/mylar3/blob/master/requirements.txt](https://github.com/mylar3/mylar3/blob/master/requirements.txt). In the `requirements.txt` below, I have omitted `unrar` and `unrar-cffi` from the original file. This is because `unrar` broke `unrar-cffi` (and we've already installed `unrar`).

Create a `requirements.txt` file:
```bash
(mylar) [root@Mylar3-1 /usr/local/virtual/mylar]# cd /tmp
(mylar) [root@Mylar3 /tmp]#  nano requirements.txt

#### ESSENTIAL LIBRARIES FOR MAIN FUNCTIONALITY ####
APScheduler>=3.6.3
beautifulsoup4>=4.8.2
cfscrape>=2.0.8
cheroot==8.2.1
CherryPy>=18.5.0
configparser>=4.0.2
feedparser>=5.2.1
Mako>=1.1.0
natsort>=3.5.2
Pillow>=4.2.1,~=6.2.2
portend>=2.6
pyinstaller>=3.5
pytz>=2019.3
requests>=2.22.0
simplejson>=3.17.0
six>=1.13.0
tzlocal>=2.0.0
urllib3>=1.25.7
```
Install these through the python packet manager, `pip`, in your virtual environment:
```bash
(mylar) [root@Mylar3 /tmp]# pip install -r requirements.txt 
```
I was unable to `pip install unrar-cffi`, because it is not built for *BSD (?). `unrar-cffi` is essential for ComicTagger to work properly, so lets instead build `unrar-cffi` from source:
```bash
(mylar) [root@Mylar3 /tmp]# cd /tmp/
(mylar) [root@Mylar3 /tmp]# wget https://files.pythonhosted.org/packages/32/6b/5f6cffd8e30304d160933342214c097bb7dca9d52bd6cf14a1678b2ea0b9/unrar-cffi-0.1.0a5.tar.gz
(mylar) [root@Mylar3 /tmp]# tar -xvzf unrar-cffi-0.1.0a5.tar.gz
(mylar) [root@Mylar3 /tmp]# cd unrar-cffi-0.1.0a5
```
Edit `buildconf.py` and change `[getenv("MAKE", 'make')` to `[getenv("MAKE", 'gmake')``
```bash
(mylar) [root@Mylar3 /tmp/unrar-cffi-0.1.0a5]# nano buildconf.py
(..)
BUILD_CMD = [getenv("MAKE", 'gmake'), "-C", UNRARSRC, "lib"]
```
Save, then build and install:
```bash
(mylar) [root@Mylar3 /tmp/unrar-cffi-0.1.0a5]# python setup.py build
(mylar) [root@Mylar3 /tmp/unrar-cffi-0.1.0a5]# python setup.py install
```

To see if the package has been installed, open `python`: 
```bash
(mylar) [root@Mylar3 /tmp/unrar-cffi-0.1.0a5]# python
Python 3.7.6 (default, Jan 30 2020, 01:17:40) 
[Clang 8.0.0 (tags/RELEASE_800/final 356365)] on freebsd11
Type "help", "copyright", "credits" or "license" for more information.
>>> import unrar.cffi
>>>
>>> exit() 
(mylar) [root@Mylar3 /tmp/unrar-cffi-0.1.0a5]#
```
If it comes back with an error, it's not installed. If it just goes to the next line, it's installed.

# Install mylar3
Now since we have all the required dependencies, let us install Mylar.

We'll use `git clone` and this will create a folder called `mylar3` in the working current directory. Let us do this under the `/usr/local/` folder:
```bash
(mylar) [root@Mylar3 /tmp/unrar-cffi-0.1.0a5]# cd /usr/local
(mylar) [root@Mylar3 /usr/local]$ git clone https://github.com/mylar3/mylar3.git
```
# Configuration of mylar3
## chown
Make our service user `mylar` the owner of the newly created `mylar3` folder, and do this recursively:
```bash
(mylar) [root@Mylar3 /usr/local]$ chown -R mylar mylar3/
```

## Carepackage
If you try to generate a Carepackage (for logs and whatnot), it will fail. We have to create a symbolic link for `pip3` first.
```bash
500 Internal Server Error

The server encountered an unexpected condition which prevented it from fulfilling the request.

Traceback (most recent call last):
  File "/usr/local/lib/python3.7/site-packages/cherrypy/_cprequest.py", line 670, in respond
    response.body = self.handler()
  File "/usr/local/lib/python3.7/site-packages/cherrypy/lib/encoding.py", line 217, in __call__
    self.body = self.oldhandler(*args, **kwargs)
  File "/usr/local/lib/python3.7/site-packages/cherrypy/_cpdispatch.py", line 60, in __call__
    return self.callable(*self.args, **self.kwargs)
  File "/usr/local/mylar3/mylar/webserve.py", line 5615, in carepackage
    cp = carePackage()
  File "/usr/local/mylar3/mylar/carepackage.py", line 25, in __init__
    self.environment()
  File "/usr/local/mylar3/mylar/carepackage.py", line 60, in environment
    text=True)
  File "/usr/local/lib/python3.7/subprocess.py", line 488, in run
    with Popen(*popenargs, **kwargs) as process:
  File "/usr/local/lib/python3.7/subprocess.py", line 800, in __init__
    restore_signals, start_new_session)
  File "/usr/local/lib/python3.7/subprocess.py", line 1551, in _execute_child
    raise child_exception_type(errno_num, err_msg, err_filename)
FileNotFoundError: [Errno 2] No such file or directory: 'pip3': 'pip3'
Powered by CherryPy 5.4.0
```
### Symbolic link pip
```bash
(mylar) [root@Mylar3 /tmp]# ln -s /usr/local/bin/pip-3.7 /usr/local/bin/pip3
```

# Manual start
If you would like to manually start `mylar3` with the user `mylar` and all the default settings, I suggest doing this:
```bash
(mylar) [root@Mylar3 /tmp]# screen
root@Mylar3:/tmp # bash
(mylar) [root@Mylar3 /tmp]# source /usr/local/virtual/mylar/bin/activate
(mylar) [root@Mylar3 /tmp]# sudo -u mylar -s bash
(mylar) (mylar) [mylar@Mylar3 /tmp]$ python /usr/local/mylar3/Mylar.py 
```

Use CTRL + A + D to detach from screen and `mylar` will be running in the "background".

You will now be able to access `mylar3`, if you open http://ip-adress:8090/home.

To reattach the `screen` session, write `screen -r`. For now, stop Mylar with using CTRL + C. You can exit from the screen session, writing `exit`, `exit` and `exit`. 

# Automatic start
To make `mylar3` start at boot of the jail, do the following:
## rc.d
Create a file called `mylar3` in `/usr/local/etc/rc.d/`:
```bash
root@Mylar3:/usr/local/bin # nano /usr/local/etc/rc.d/mylar3

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
# mylar3_user:  The user account Mylar daemon runs as what
#           you want it to be. It uses 'mylar' user by
#           default. Do not sets it as empty or it will run
#           as root.
# mylar3_dir:   Directory where Mylar lives.
#           Default: /usr/local/mylar3
# mylar3_pid:  The name of the pidfile to create.
#           Default is mylar.pid in mylar_dir.
# mylar3_conf: The name of the config file you would like to launch with mylar3.
#
# command:    The path to your virtual environment executable
#
# https://psychogun.github.io/docs/freenas/Mylar3-in-a-FreeNAS-iocage-jail/
#

. /etc/rc.subr

name="mylar3"
rcvar="${name}_enable"

load_rc_config ${name}
: "${mylar3_enable:="NO"}"
: "${mylar3_user:="mylar"}"
: "${mylar3_dir:="/usr/local/mylar3"}"
: "${mylar3_conf:="/usr/local/mylar3/config.ini"}"

pidfile="/var/run/mylar3/mylar3.pid"

command="/usr/local/virtual/mylar/bin/python"
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
root@Mylar3:/usr/local/bin # chmod +x /usr/local/etc/rc.d/mylar3 
```
Add the following line to /etc/rc.conf.local or /etc/rc.conf to enable this service:
```bash
root@Mylar3:/usr/local/bin # nano /etc/rc.conf

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

# Fault finding
## pip freeze
On the jail:
```bash
root@Mylar3:~ # pip freeze
sqlite3==0.0.0
virtualenv==16.7.5
root@Mylar3:~ # 
```
In the virtual environment:
```bash
root@Mylar3:~ # sudo -u mylar -s bash
[mylar@Mylar3 /root]$ source /usr/local/virtual/mylar/bin/activate
(mylar) [mylar@Mylar3 /root]$ pip freeze
altgraph==0.17
APScheduler==3.6.3
beautifulsoup4==4.8.2
certifi==2019.11.28
cffi==1.14.0
cfscrape==2.1.1
chardet==3.0.4
cheroot==8.2.1
CherryPy==18.5.0
configparser==5.0.0
feedparser==5.2.1
idna==2.9
jaraco.classes==3.1.0
jaraco.collections==3.0.0
jaraco.functools==3.0.0
jaraco.text==3.2.0
Mako==1.1.2
MarkupSafe==1.1.1
more-itertools==8.2.0
natsort==7.0.1
Pillow==6.2.2
portend==2.6
pycparser==2.20
PyInstaller==3.6
pytz==2019.3
requests==2.23.0
simplejson==3.17.0
six==1.14.0
soupsieve==2.0
sqlite3==0.0.0
tempora==3.0.0
tzlocal==2.0.0
unrar-cffi==0.1.0a5
urllib3==1.25.8
zc.lockfile==2.0
```

## unrar.cffi
```bash
(mylar) [mylar@Mylar3 /root]$ python
Python 3.7.6 (default, Mar 21 2020, 01:17:29) 
[Clang 8.0.0 (tags/RELEASE_800/final 356365)] on freebsd11
Type "help", "copyright", "credits" or "license" for more information.
>>> import unrar.cffi
>>> 
```

## Status
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

## Change branch
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
Save and use `service mylar3 start` to start the service.

## Debug mode
Add the option `-v` to `Mylar.py` to enable `DEBUG` logging.

```bash
root@Mylar3:~ # service mylar3 stop
root@Mylar3:~ # sudo -u mylar -s bash
[mylar@Mylar3 /root]$ source /usr/local/virtual/mylar/bin/activate
(mylar) [mylar@Mylar3 /root]$ python /usr/local/mylar3/Mylar.py -v
```
Then use `(mylar) [mylar@Mylar3 /root]$ tail -f /usr/local/mylar3/logs/mylar.log`.

## Migrate 
### /cache
```bash
root@Freenas:~ # cp -rp /mnt/TankJr/iocage/jails/Mylar3/root/usr/local/mylar3/cache/* /mnt/TankJr/iocage/jails/Mylar3-1/root/usr/local/mylar3/cache/
```
### config.ini
```bash
root@Freenas:~ # cp -p /mnt/TankJr/iocage/jails/Mylar3/root/usr/local/mylar3/config.ini /mnt/TankJr/iocage/jails/Mylar3-1/root/usr/local/mylar3/
```
### mylar.db
```bash
root@Freenas:~ # cp -p /mnt/TankJr/iocage/jails/Mylar3/root/usr/local/mylar3/mylar.db /mnt/TankJr/iocage/jails/Mylar3-1/root/usr/local/mylar3/
```

## Authors
Mr. Johnson

## Acknowledgments
* [https://docs.python.org/3/library/venv.html](https://docs.python.org/3/library/venv.html)
* [https://computingforgeeks.com/how-to-install-pip-python-package-manager-on-freebsd-12/](https://computingforgeeks.com/how-to-install-pip-python-package-manager-on-freebsd-12/)
* [https://www.freebsd.org/doc/en_US.ISO8859-1/articles/rc-scripting/](https://www.reddit.com/r/nzbhydra/comments/8fqcyl/freebsd_install_guide/)
* [https://www.freebsd.org/cgi/man.cgi?pw(8)](https://www.freebsd.org/cgi/man.cgi?pw(8))
* [https://forums.freebsd.org/threads/pkg-quarterly-to-latest.72196/](https://forums.freebsd.org/threads/pkg-quarterly-to-latest.72196/)


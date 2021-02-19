---
layout: default
title: How to install AirDC++ in a FreeNAS iocage jail
parent: FreeNAS
nav_order: 1
---

# How to install AirDC++ in a FreeNAS iocage jail
{: .no_toc }
This is how i installed a communal peer-to-peer file sharing application for file servers/NAS devices called AirDC++ on FreeNAS in a standalone iocage jail.

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
These instructions will download, compile, install and run AirDC++ in an iocage jail on FreeNAS. There are no provided binaries for BSD on [https://airdcpp-web.github.io/docs/installation/installation.html](https://airdcpp-web.github.io/docs/installation/installation.html).

### Prerequisites
* Knowledge of SSH and how to navigate to your jail in FreeNAS
* FreeNAS 11.2 and knowledge of how to create a jail with shares and knowledge of UNIX folder and files permissions

---

## Compiling
Update and upgrade your iocage jail first:
```bash
root@Airdccp:/ # pkg upgrade && pkg update
```

To compile Airdcpp on FreeNAS (FreeBSD), we need to install all the required dependencies listed here
[https://airdcpp-web.github.io/docs/installation/dependencies.html](https://airdcpp-web.github.io/docs/installation/dependencies.html):
```bash
root@Airdccp:/ # pkg install gcc cmake pkgconf npm node python boost-all bzip2 leveldb miniupnpc openssl websocketpp tbb php72-maxminddb git nano screen
```

Use git clone to download AirDC++ in an appropiate folder:
```bash
root@Airdccp:/ # cd /usr/local/
root@Airdccp:/usr/local# git clone https://github.com/airdcpp-web/airdcpp-webclient.git
```

Hop on in to the newly created folder and call on `cmake .` to generate a makefile:
```bash
root@Airdccp:/usr/local # cd airdcpp-webclient/
root@Airdccp:/usr/local/airdcpp-webclient # cmake .
```

Use `make` to compile AirDC++ with gcc (the number after -j indicates how many processor cores you are giving the task):
```bash
root@Airdccp:/usr/local/airdcpp-webclient # make -j6
```
---

## Add a user
Add a user to run AirDC++, we do not want AirDC++ to run as root:
```
root@Airdccp:/usr/local/airdcpp-webclient # adduser
Username: zanko
Full name: Airdcpp user
Uid (Leave empty for default): 
Login group [zanko]: 
Login group is zanko. Invite zanko into other groups? []: 
Login class [default]: 
Shell (sh csh tcsh git-shell nologin) [sh]: nologin
Home directory [/home/zanko]: 
Home directory permissions (Leave empty for default): 
Use password-based authentication? [yes]: 
Use an empty password? (yes/no) [no]: 
Use a random password? (yes/no) [no]: yes
Lock out the account after creation? [no]: 
Username   : zanko
Password   : <random>
Full Name  : Airdcpp user
Uid        : 1001
Class      : 
Groups     : zanko 
Home       : /home/zanko
Home Mode  : 
Shell      : /usr/sbin/nologin
Locked     : no
OK? (yes/no): yes
adduser: INFO: Successfully added (zanko) to the user database.
adduser: INFO: Password for (zanko) is: 
Add another user? (yes/no): no
```
---

## Install
```bash
root@Airdccp:/usr/local/airdcpp-webclient # make install
```
First time you are using AirDC++, you have to configure it:
```bash
root@Airdcpp:/usr/local/airdcpp-webclient # airdcppd/airdcppd --configure
```
The `--configure` parameter will create a config file, `WebServer.xml`, under your root home folder which we will copy to our created user' home space:
```bash
root@Airdcpp:/usr/local/airdcpp-webclient # cp /root/.airdc++/WebServer.xml /home/zanko/.airdc++/WebServer.xml
```
Change ownership of the .airdc++ folder which we just created, recursively (-R):
```bash
root@Airdcpp:/usr/local/airdcpp-webclient # chown -R zanko:zanko /home/zanko/.airdc++/
```

### Manual run
Check if you are able to run AirDC++ as our newly created user: (more about the command line options can be found here: [https://airdcpp-web.github.io/docs/usage/command-line-options.html](https://airdcpp-web.github.io/docs/usage/command-line-options.html)
Basically, what you now want to do is open `screen` and run the command below and then detach (ctrl + a, d) from the `screen` session.
```
root@Airdcpp:/usr/local/airdcpp-webclient # screen
root@Airdcpp:/usr/local/airdcpp-webclient # su -m zanko -c '/usr/local/airdcpp-webclient/airdcppd/airdcppd -c=/home/zanko/.airdc++/'
```
Use the command 'top' to see if airdcpp is running as our user zanko:
```
root@Airdccp:/ # top
 PID USERNAME    THR PRI NICE   SIZE    RES STATE   C   TIME    WCPU COMMAND
 1000 zanko         13  52   19 57528K 28520K uwait  10   0:01   0.05% airdcppd
```
Hit ctrl + c to close `top`.
To connect to your session again, write `screen -r`.

### Automatic run
Create a file called `airdcppd` in `/usr/local/etc/rc.d/`:
```bash
root@Airdcpp:/usr/local/bin # nano /usr/local/etc/rc.d/airdcppd

#!/bin/sh
#
# PROVIDE: airdcpp
# REQUIRE: DAEMON
# BEFORE:  LOGIN
# KEYWORD: shutdown
#
# Add the following line to /etc/rc.conf.local or /etc/rc.conf
# to enable this service:
#           airdcppd_enable="YES"
# Start it with:
#           service airdcppd start
#
# Stop it with:
#           service airdcppd stop
#
#
#
#
# zanko:  The user account airdcppd daemon runs as what
#           you want it to be. It uses 'zanko' user by
#           default. Do not sets it as empty or it will run
#           as root.
# airdcpp_dir:   Directory where airdccpd lives.
#           Default: /usr/local/airdcpp-webclient/airdcppd
# airdcpp_pid:  The name of the pidfile to create.
#               Custom pid file path (default: /.airdcppd.pid)
#
# airdcpp_conf: The directory of where WebServer.xml resides
#
#
# https://psychogun.github.io/docs/freenas/Airdcpp-in-a-FreeNAS-iocage-jail/
#

. /etc/rc.subr

name="airdcppd"
rcvar="${name}_enable"
pidfile="/var/run/${name}/${name}.pid"

load_rc_config ${name}
: "${airdcppd_enable:="NO"}"
: "${airdcppd_user:="zanko"}"
: "${airdcppd_dir:="/usr/local/airdcpp-webclient/airdcppd"}"
: "${airdcppd_conf:="/home/zanko/.airdc++/"}"


command="$airdcppd_dir/airdcppd"
command_args="-d -p=$pidfile -c=$airdcppd_conf"


start_precmd="${name}_start_precmd"
airdcppd_start_precmd() {
        if [ $($ID -u) != 0 ]; then
                err 1 "Must be root."
        fi

        if [ ! -d /var/run/$name ]; then
                install -do $airdcppd_user /var/run/$name
        fi
}

load_rc_config ${name}
run_rc_command "$1"

```
Make the file executable:
```bash
root@Airdcpp:/usr/local/etc/rc.d # chmod +x /usr/local/etc/rc.d/airdcppd 
```

Add the following line to `/etc/rc.conf` to enable this service:
```bash
root@Airdcpp:/usr/local/bin # nano /etc/rc.conf

(...)
# Enable Airdcppd
airdcppd_enable="YES"
```

Start the `airdcppd` daemon:
```bash
root@Airdcpp:/usr/local/etc/rc.d # service airdcppd start
```

---

## Add share

---

## getfacl
Create a group called dump with GID = 1049 in the iocage jail.
```bash
root@Airdcpp:~ # pw groupadd dump -g 1049
root@Airdcpp:~ # groups zanko
zanko
root@Airdcpp:~ # pw usermod zanko -G dump
root@Airdcpp:~ # groups zanko
zanko dump
root@Airdcpp:~ # 
```

---

## Authors
Mr. Johnson

---

## Acknowledgments
* [https://www.reddit.com/r/freenas/comments/9ijrsp/ive_read_dozens_of_guides_but_i_still_dont/](https://www.reddit.com/r/freenas/comments/9ijrsp/ive_read_dozens_of_guides_but_i_still_dont/)
* [https://groups.google.com/forum/#!topic/openhab/o1FIhkvSq90](https://groups.google.com/forum/#!topic/openhab/o1FIhkvSq90)
* [https://airdcpp-web.github.io/docs/usage/command-line-options.html](https://airdcpp-web.github.io/docs/usage/command-line-options.html)


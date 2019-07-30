---
layout: default
title: How to install Airdcpp in a FreeNAS iocage jail
parent: FreeNAS
nav_order: 1
---

# How to install Airdcpp in a FreeNAS iocage jail
{: .no_toc }
This is how i installed a communal peer-to-peer file sharing application for file servers/NAS devices called Airdcpp on FreeNAS in a standalone iocage jail.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}
---
## Getting started
These instructions will download, compile, install and run Airdcpp in an iocage jail on FreeNAS. There are no provided binaries for BSD on https://airdcpp-web.github.io/docs/installation/installation.html](https://airdcpp-web.github.io/docs/installation/installation.html)

### Prerequisites
* Knowledge of SSH and how to navigate to your jail in FreeNAS
* FreeNAS 11.2 and knowledge of how to create a jail with shares and knowledge of UNIX folder and files permissions

## Compiling
Update and upgrade your iocage jail first:
```tcsh
root@Airdccp:/ # pkg upgrade && pkg update
```
To compile Airdcpp on FreeNAS (FreeBSD), we need to install all the required dependencies: 
https://airdcpp-web.github.io/docs/installation/dependencies.html
```bash
root@Airdccp:/ # pkg install gcc cmake pkgconf npm node python boost-all bzip2 leveldb miniupnpc openssl websocketpp tbb php72-maxminddb git nano 
```

Use git clone to download airdcpp:
```sh
root@Airdccp:/ # cd /usr/local/
root@Airdccp:/usr/local# git clone [https://github.com/airdcpp-web/airdcpp-webclient.git](https://github.com/airdcpp-web/airdcpp-webclient.git)

root@Airdccp:/usr/local # cd airdcpp-webclient/
root@Airdccp:/usr/local/airdcpp-webclient # cmake .
```

Make, using gcc to compile airdcpp. The number after -j indicates how many processor cores you are giving the task:
```
root@Airdccp:/usr/local/airdcpp-webclient # make -j6
```
## Add a user
Add a user to run airdcpp, we do not want airdcpp to run as root:
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
## Install
Install it:
```
root@Airdccp:/usr/local/airdcpp-webclient # make install
```
First time you are using airdcpp, you have to configure it:
```
root@Airdcpp:/usr/local/airdcpp-webclient # airdcppd/airdcppd --configure
```
The --configure option will create a config file, WebServer.xml, under your root home folder which we will copy to our created user' home space:
```
root@Airdcpp:/usr/local/airdcpp-webclient # cp /root/.airdc++/WebServer.xml /home/zanko/.airdc++/WebServer.xml
```
Change ownership of folder which we just created, recursively (-R):
```
root@Airdcpp:/usr/local/airdcpp-webclient # chown -R zanko:zanko /home/zanko/.airdc++/
```
Check if you are able to run airdcpp as your user: (more about the command line options can be found here: https://airdcpp-web.github.io/docs/usage/command-line-options.html)
```
root@Airdcpp:/usr/local/airdcpp-webclient # su -m zanko -c '/usr/local/airdcpp-webclient/airdcppd/airdcppd -c=/home/zanko/.airdc++/'

```
Use the command 'top' to see if airdcpp is running as our user zanko:
```
root@Airdccp:/ # top
 PID USERNAME    THR PRI NICE   SIZE    RES STATE   C   TIME    WCPU COMMAND
 1000 zanko         13  52   19 57528K 28520K uwait  10   0:01   0.05% airdcppd
```
There you have it. 

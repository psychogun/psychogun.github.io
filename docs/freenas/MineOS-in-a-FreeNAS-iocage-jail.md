---
layout: default
title: How to install MineOS in a FreeNAS iocage jail
parent: FreeNAS
nav_order: 9
---

# How to install MineOS in a FreeNAS iocage jail
{: .no_toc }
From 

## Table of contents
{: .no_toc .text-delta }
1. TOC
{:toc}
---

## Getting started
Go to your FreeNAS. Select Jails > and click Add.
Jail Name: Lidarr
Release: 11.2-RELEASE

### Shell into your jail & install the necessary packages
```bash
root@MineOS:~ # pkg update
The package management tool is not yet installed on your system.
Do you want to fetch and install it now? [y/N]: y 


root@MineOS:~ # pkg upgrade
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
Updating database digests format: 100%
Checking for upgrades (1 candidates): 100%
Processing candidates (1 candidates): 100%
Checking integrity... done (0 conflicting)
Your packages are up to date.
root@MineOS:~ # 
```

```bash
root@MineOS:~ # pkg install -y rdiff-backup rsync gmake screen git python2 sysutils/py-supervisor node8 npm-node8 openjdk8-jre wget bash
```

### Install MineOS-node
```bash
root@MineOS:~ # mkdir -p /usr/compat/linux/proc
root@MineOS:~ # mkdir -p /usr/local/games
root@MineOS:~ # cd /usr/local/games/
root@MineOS:/usr/local/games # git clone git://github.com/hexparrot/mineos-node minecraft
Cloning into 'minecraft'...
remote: Enumerating objects: 5862, done.
remote: Total 5862 (delta 0), reused 0 (delta 0), pack-reused 5862
Receiving objects: 100% (5862/5862), 5.03 MiB | 5.78 MiB/s, done.
Resolving deltas: 100% (3447/3447), done.
root@MineOS:/usr/local/games # cd minecraft/
root@MineOS:/usr/local/games/minecraft # chmod +x *.sh
root@MineOS:/usr/local/games/minecraft # ./generate-sslcert.sh 
Generating a 1024 bit RSA private key
................++++++
................++++++
writing new private key to '.tmpkey.pem'
-----
writing RSA key
root@MineOS:/usr/local/games/minecraft # cp mineos.conf /etc/mineos.conf
root@MineOS:/usr/local/games/minecraft # echo "CXX=c++ npm install" | sh

> posix@4.1.2 install /usr/local/games/minecraft/node_modules/posix
> node-gyp rebuild

gmake: Entering directory '/usr/local/games/minecraft/node_modules/posix/build'
  CXX(target) Release/obj.target/posix/src/posix.o
  SOLINK_MODULE(target) Release/obj.target/posix.node
  COPY Release/posix.node
gmake: Leaving directory '/usr/local/games/minecraft/node_modules/posix/build'

> userid@0.3.1 install /usr/local/games/minecraft/node_modules/userid
> node-gyp rebuild

gmake: Entering directory '/usr/local/games/minecraft/node_modules/userid/build'
  CXX(target) Release/obj.target/userid/src/userid.o
  SOLINK_MODULE(target) Release/obj.target/userid.node
  COPY Release/userid.node
gmake: Leaving directory '/usr/local/games/minecraft/node_modules/userid/build'
added 568 packages from 431 contributors and audited 1309 packages in 18.885s
found 21 vulnerabilities (8 low, 4 moderate, 9 high)
  run `npm audit fix` to fix them, or `npm audit` for details
root@MineOS:/usr/local/games/minecraft # 
```

#### Run npm audit fix
```bash
root@MineOS:/usr/local/games/minecraft # npm audit fix
+ angular@1.7.9
+ angular-translate@2.18.1
updated 4 packages in 4.068s
fixed 8 of 21 vulnerabilities in 1309 scanned packages
  3 vulnerabilities required manual review and could not be updated
  1 package update for 10 vulnerabilities involved breaking changes
  (use `npm audit fix --force` to install breaking changes; or refer to `npm audit` for steps to fix these manually)
root@MineOS:/usr/local/games/minecraft # 
```

### Startup MineOS-node when jail starts
```bash
root@MineOS:/usr/local/games/minecraft # cat /usr/local/games/minecraft/init/supervisor_conf.bsd >> /usr/local/etc/supervisord.conf
root@MineOS:/usr/local/games/minecraft # echo 'supervisord_enable="YES"' >> /etc/rc.conf
```

### 6. Exit the jail & go back to FreeNAS webUI
a) Stop the jail

Edit > Custom Properties:
* mount_linprocfs

SAVE


### Add a unprivileged user to use with the MineOS web-ui
```bash
root@MineOS:/usr/local/games/minecraft # adduser
Username: mcserver
Full name: User for MineOS
Uid (Leave empty for default): 
Login group [mcserver]: 
Login group is mcserver. Invite mcserver into other groups? []: games
Login class [default]: 
Shell (sh csh tcsh git-shell bash rbash nologin) [sh]: bash
Home directory [/home/mcserver]: 
Home directory permissions (Leave empty for default): 
Use password-based authentication? [yes]: 
Use an empty password? (yes/no) [no]: 
Use a random password? (yes/no) [no]: yes
Lock out the account after creation? [no]: 
Username   : mcserver
Password   : <random>
Full Name  : User for MineOS
Uid        : 1001
Class      : 
Groups     : mcserver games
Home       : /home/mcserver
Home Mode  : 
Shell      : /usr/local/bin/bash
Locked     : no
OK? (yes/no): yes
adduser: INFO: Successfully added (mcserver) to the user database.
adduser: INFO: Password for (mcserver) is: uvIuLazz98R
Add another user? (yes/no): no
Goodbye!
```

Running the server for the first time
In this section, you’ll perform the following tasks:

Download the Minecraft server binaries.
Accept the Minecraft end user license agreement (EULA).
Start the server.
To download the Minecraft server binaries:

## Configure
### Spigot
Press "Download latest BuildTools.jar"

Then press "Build Spigot" to the right of the version you want (1.15.1). You might have to refresh your webbrowser. It is buildt when you have "Delete" and "Copy to server" buttons. 

### Profiles

### Create New Server
Create a new server. 

Then go to Spigot. Select your server. Press Copy to Server
Copy compiled binaries to server directory, and select your server. Click "Copy to server".


### Dashboard
Go to Dashboard, select your newly created server.

1. Click Start.
1.1. Change profile to: 1.15.1
1.2. Change runnable jar to: minecraft_server.1.15.1.jar

Memory Allocation (Heapsize)


* Broadcast to LAN
* Start server on boot

+ Create a new restore point

### Server settings
#### Server.properties

enforce-whitelist: true
online-mode: true

#### Whitelist
Go to LOGGING > logs/latest.log

Add users by typing next to the `>`:

### Change VLAN
My game servers are on VLAN 120.

This VLAN is sent to my interface (bridge0), amongst other VLANS. 

Edit the jail, go to Network Properties and select `interfaces vnet0:bridge1`, change that to `vnet0:bridge0`.
Click SAVE.

Now the jail will boot up with the network that has VLANs. But it will only have the ipaddress of the untagged network. 

Check what your interface is called by issuing `ifconfig`:
```bash
root@MineOS:~ # ifconfig
epair0b: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500

(...)
```


Edit `rc.conf`:
```bash
root@MineOS:~ # nano /etc/rc.conf
# Enable VLAN
vlans_epair0b="120"
ifconfig_epair0b_120="DHCP"
```
Stop and start the MineOS jail.

Issue command for starting DHCP on the VLAN 120 interface:
```bash
root@MineOS:~ # dhclient epair0b.120
```


## Create VLAN
### ADD VLAN
FreeNAS gui > Network > VLANs. 
Click ADD. 

Virtual Interface: vlan120
Parent Interface: em1 (the physical interface where tagged VLAN traffic is coming)
Vlan Tag: 120
Description: Game Servers
Priority Code Point: Best effort (default)


### Define cloned interfaces / create a bridge
Go to FreeNAS gui > System > Tunables.
Click ADD.

Bridge0 is my bridge where I receive the VLANs. 

Variable: cloned_interfaces
Value: bridge0 bridge120
Type: rc
Comment: Setup bridges
Enabled: Yes


### Add VLAN to your created bridge120
Go to FreeNAS gui > System > Tunables.
Click ADD. 

Variable: ifconfig_bridge120
Value: addm vlan140 up
Type: rc
Comment: VLAN 120 (Game servers)

### Edit jail
Edit > Network properties

interfaces
vnet0:bridge120

### Configure jail's default gateway

```bash
root@freenas:~ # iocage set defaultrouter=192.168.120.1 MineOS
```


## Fault finding
### Update / reset MineOS webgui
```bash
root@MineOS:/var/games/minecraft # cd /usr/local/games/minecraft
root@MineOS:/usr/local/games/minecraft # git fetch
root@MineOS:/usr/local/games/minecraft # git reset --hard origin/master
HEAD is now at d743d22 adding test confirming previous commit
root@MineOS:/usr/local/games/minecraft # rm -rf node_modules
root@MineOS:/usr/local/games/minecraft # echo "CXX=c++ npm install" | sh
```

## Authors
Mr. Johnson


## Acknowløedgements
* [https://devpro.media/minecraft-server-freenas/#preparing-the-jail](https://devpro.media/minecraft-server-freenas/#preparing-the-jail)
* [https://www.freebsd.org/doc/handbook/network-vlan.html](https://www.freebsd.org/doc/handbook/network-vlan.html)
* [https://www.freebsd.org/doc/handbook/config-network-setup.html](https://www.freebsd.org/doc/handbook/config-network-setup.html)
* [https://forums.freebsd.org/threads/command-for-starting-dhcp.48216/](https://forums.freebsd.org/threads/command-for-starting-dhcp.48216/)
* [https://forums.freebsd.org/threads/cant-assign-ip-addresses-to-vlan-interfaces-in-rc-conf.66454/](https://forums.freebsd.org/threads/cant-assign-ip-addresses-to-vlan-interfaces-in-rc-conf.66454/)
* [https://forums.freebsd.org/threads/solved-how-to-skip-wifi-bring-up-during-boot.44956/#post-250815](https://forums.freebsd.org/threads/solved-how-to-skip-wifi-bring-up-during-boot.44956/#post-250815)
* [https://discourse.codeemo.com/t/how-to-white-list-people-for-minecraft/2754](https://discourse.codeemo.com/t/how-to-white-list-people-for-minecraft/2754
* [https://minecraft.gamepedia.com/Commands/whitelist](https://minecraft.gamepedia.com/Commands/whitelist)
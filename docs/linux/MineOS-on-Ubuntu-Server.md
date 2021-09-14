---
layout: default
title: How to install MineOS on Ubuntu Server
parent: Linux
nav_order: 14
---
# How to install MineOS  on Ubuntu Server
{: .no_toc }
This is how I created a MineOS server for multiplaying on my network. Mostly based on this guide [https://minecraft.codeemo.com/mineoswiki/index.php?title=MineOS-node_(apt-get%2Bsystemd)](https://minecraft.codeemo.com/mineoswiki/index.php?title=MineOS-node_(apt-get%2Bsystemd)), but some alteratings for my setup.

* Your webui for managing your servers will ultimately be on https://xxx.yyy.zzz.aaa:8443

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


### Prerequisites
* Proxmox 6.1
* Ubuntu 20.04
* Mojang 16.1.3

### Set your time zone
```bash
per@sson:~$ date
Sat 11 Jan 21:22:53 GMT 2020
per@sson:~$ sudo dpkg-reconfigure tzdata

Current default time zone: 'Europe/Paris'
Local time is now:      Sat Jan 11 22:24:07 CET 2020.
Universal Time is now:  Sat Jan 11 21:24:07 UTC 2020.

per@sson:~$ $ date
Sat 11 Jan 22:24:16 CET 2020
```

### Disable IPv6
```bash
per@sson:~$ sudo nano /etc/default/grub
(...)
GRUB_CMDLINE_LINUX_DEFAULT="maybe-ubiquity"
GRUB_CMDLINE_LINUX=""
```
Change to:
```bash
(...)
GRUB_CMDLINE_LINUX_DEFAULT="maybe-ubiquity ipv6.disable=1"
GRUB_CMDLINE_LINUX="ipv6.disable=1"
```
Then run:
```bash
per@sson:~$ sudo update-grub
```

### Change NTP server
```bash
per@sson:~$ nano /etc/systemd/timesyncd.conf 
(...)
[Time]
#NTP=
(...)
```
Change to
```bash
(...)
[Time]
NTP=192.168.44.1 pool.ntp.org
(...)
```
Restart NTP service
```bash
per@sson:~$ sudo systemctl restart systemd-timesyncd
```

### qemu-guest-agent
```bash
per@sson:~$ sudo apt update
per@sson:~$ sudo apt upgrade
per@sson:~$ sudo apt install qemu-guest-agent
```
Shut down the VM. Enable Qemu Guest Agent in Proxmox and start the VM. 

---

## Installation of MineOS
Does the installation come with `java`?
```
per@sson:~$ java -version

Command 'java' not found, but can be installed with:

sudo apt install openjdk-11-jre-headless  # version 11.0.8+10-0ubuntu1~20.04, or
sudo apt install default-jre              # version 2:1.11-72
sudo apt install openjdk-13-jre-headless  # version 13.0.3+3-1ubuntu2
sudo apt install openjdk-14-jre-headless  # version 14.0.1+7-1ubuntu1
sudo apt install openjdk-8-jre-headless   # version 8u265-b01-0ubuntu2~20.04

per@sson:~$ 
```
Allright, no biggie; we'll install it in the next section.

### Prerequsities
```bash
per@sson:~$ sudo -i
sudo curl -sL https://deb.nodesource.com/setup_10.x | bash -
sudo apt-get install -y nodejs
```

```bash
apt-get update
sudo apt-get -y install -y git rdiff-backup screen build-essential openjdk-8-jre-headless
```
### Folder creations:
```bash
per@sson:~$ mkdir -p /usr/games
per@sson:~$ cd /usr/games
per@sson:/usr/games$ sudo git clone https://github.com/hexparrot/mineos-node.git minecraft
per@sson:/usr/games$ cd minecraft
per@sson:/usr/games$ sudo git config core.filemode false
per@sson:/usr/games$ sudo chmod +x service.js mineos_console.js generate-sslcert.sh webui.js
per@sson:/usr/games$ sudo npm install --unsafe-perm
per@sson:/usr/games$ sudo ln -s /usr/games/minecraft/mineos_console.js /usr/local/bin/mineos
per@sson:/usr/games$ sudo cp mineos.conf /etc/mineos.conf
```

### Running the MineOS Web Service
Have the web interface start at boot:

```bash
per@sson:/usr/games$ sudo cp init/systemd_conf /etc/systemd/system/mineos.service
per@sson:/usr/games$ systemctl enable mineos
```

#### Secure HTTPS operation
Before you can start the server, you must generate a self-signed certificate for HTTPS functionality. Using this script, you can generate a `mineos.key` / `mineos.crt` / `mineos.pem` file located in `/etc/ssl/certs/`:

```bash
per@sson:/usr/games$ sudo ./generate-sslcert.sh
```

### chown -R
```bash
per@sson:/usr/games$ sudo chown -R mine minecraft/
```

### Manual start
Remember, you won't need to do this on subsequent restarts, as the initscript will take care of it.
```bash
per@sson:/usr/games$ systemctl start mineos
```

### Manual stop
```bash
per@sson:/usr/games$ systemctl stop mineos
```

Using the webui
Overview
The scripts, by default, will run a server operating on port 8443 and place minecraft data files into `/var/games/minecraft`.

When creating minecraft servers, it is required to use an unprivileged user to create and manage Minecraft servers. For most distros, this will be with the `adduser username` command. The password you set during user creation will also be the password used for the web-ui.

In your browser, visit the location: https://xxx.yyy.zzz.aaa:8443

---

## Creating servers
Servers may only be created by unprivileged users, or in other words: not root. Be sure to log in as any unprivileged user to create any servers you wish and leverage group membership to share control of servers with others!

### Profiles
* ID: Mojang
* 1.16.3 Official Mojang rar

Click Download

### Spigot
* Press "Download latest BuildTools.jar"

* Then press "Build Spigot" to the right of the version you want (1.16.3).

You might have to refresh your webbrowser. It is built when you have "Delete" and "Copy to server" buttons. 

### Create New Server
Create a new server; the naming convention is how I keep track of my servers.

* Server Name: mc_creative_normal_structures_flat
* Port: 25565


Then go to Spigot. Select your server. Press `Copy` to copy compiled binaries to the server directory.

### Dashboard
Go to Dashboard, select your newly created server.

* Memory Allocation (Heapsize): 2048 (-XMx, -XMs)

* Broadcast to LAN
* Start server on boot

* Create a new restore point
* Create a new archive

* Click Start.
* Change profile to: 1.16.3
* Change runnable jar to: minecraft_server.1.16.3.jar

Accept EULA, click Start.

---

## Server settings
### Server.properties
enforce-whitelist: true
online-mode: true

### Whitelist
Go to LOGGING > logs/latest.log and select your server.

Add users by typing next to the `>`: whitelist add [username]

---

## Fault finding

### Upgrade JAVA to play v1.17+
Unable to Build Spigot? Because you are moving to MineCraft v1.17? And you have the wrong JAVA version? 

```bash
tail -f /var/log/mineos.log
{"level":"error","message":"stderr: Error: Invalid or corrupt jarfile /var/games/minecraft/profiles/spigot_1.17.1/BuildTools.jar\n","timestamp":"2021-09-14T17:05:25.443Z"}````
```

Let's check the filesize:
```bash
per@sson:/var/games/minecraft/profiles$ cd BuildTools-latest/
per@sson:/var/games/minecraft/profiles/BuildTools-latest$ ls -l
total 4
-rw-rw-r-- 1 root root 16 Sep 14 19:00 BuildTools.jar
```

Let us make a backup: 
```bash
per@sson:/var/games/minecraft/profiles/BuildTools-latest$ sudo mv BuildTools.jar BuildTools.ja_r
```

Let us download a new BuildTool:
```bash
per@sson:/var/games/minecraft/profiles/BuildTools-latest$ sudo wget https://hub.spigotmc.org/jenkins/job/BuildTools/lastSuccessfulBuild/artifact/target/BuildTools.jar
```

Now we have a file which has a larger size:
```bash
per@sson:/var/games/minecraft/profiles/BuildTools-latest$ ls -l
total 4088
-rw-rw-r-- 1 root root      16 Sep 14 19:00 BuildTools.ja_r
-rw-r--r-- 1 root root 4179124 Aug 13 00:45 BuildTools.jar
```

Let us fix permissions:
```bash
per@sson:/var/games/minecraft/profiles/BuildTools-latest$ chmod 664 BuildTools.jar 
```

Now let's see what the log is telling us:
```bash
{"builder":{"id":"BuildTools-latest","time":1631639050675,"releaseTime":1631639050675,"type":"release","group":"spigot","webui_desc":"Latest BuildTools.jar for building Spigot/Craftbukkit","weight":0,"filename":"BuildTools.jar","downloaded":true,"version":0,"release_version":"","url":"https://hub.spigotmc.org/jenkins/job/BuildTools/lastSuccessfulBuild/artifact/target/BuildTools.jar","$$hashKey":"object:4021"},"version":"1.17.1","command":"build_jar","level":"info","message":"[WEBUI] Received emit command from 192.168.5.50:mine","timestamp":"2021-09-14T17:14:40.124Z"}
{"builder":{"id":"BuildTools-latest","time":1631639050675,"releaseTime":1631639050675,"type":"release","group":"spigot","webui_desc":"Latest BuildTools.jar for building Spigot/Craftbukkit","weight":0,"filename":"BuildTools.jar","downloaded":true,"version":0,"release_version":"","url":"https://hub.spigotmc.org/jenkins/job/BuildTools/lastSuccessfulBuild/artifact/target/BuildTools.jar","$$hashKey":"object:4021"},"version":"1.17.1","command":"build_jar","level":"info","message":"[WEBUI] BuildTools starting with arguments:","timestamp":"2021-09-14T17:14:40.134Z"}
{"level":"error","message":"stderr: *** The version you have requested to build requires Java versions between [Java 16, Java 17], but you are using Java 8","timestamp":"2021-09-14T17:15:02.557Z"}
{"level":"error","message":"stderr: \n","timestamp":"2021-09-14T17:15:02.558Z"}
{"level":"error","message":"stderr: *** Please rerun BuildTools using an appropriate Java version. For obvious reasons outdated MC versions do not support Java versions that did not exist at their release.","timestamp":"2021-09-14T17:15:02.558Z"}
{"level":"error","message":"stderr: \n","timestamp":"2021-09-14T17:15:02.559Z"}
{"level":"info","message":"[WEBUI] BuildTools jar compilation finished unsuccessfully in /var/games/minecraft/profiles/spigot_1.17.1","timestamp":"2021-09-14T17:15:02.573Z"}
{"level":"info","message":"[WEBUI] Buildtools used: /var/games/minecraft/profiles/spigot_1.17.1/BuildTools.jar","timestamp":"2021-09-14T17:15:02.573Z"}
```

Aha. 
* [https://www.spigotmc.org/threads/mojang-is-moving-to-java-16.505588/](https://www.spigotmc.org/threads/mojang-is-moving-to-java-16.505588/) 

We will have to update our java version:
```bash
per@sson:/var/games/minecraft/profiles$ java -version
openjdk version "1.8.0_292"
OpenJDK Runtime Environment (build 1.8.0_292-8u292-b10-0ubuntu1~20.04-b10)
OpenJDK 64-Bit Server VM (build 25.292-b10, mixed mode)
```

Let's follow this guide:
* [https://java.tutorials24x7.com/blog/how-to-install-openjdk-16-on-ubuntu-20-04-lts](https://java.tutorials24x7.com/blog/how-to-install-openjdk-16-on-ubuntu-20-04-lts)


Stop the `mineos.service`:
```
per@sson:/var/games/minecraft/profiles$ systemctl stop mineos
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-units ===
Authentication is required to stop 'mineos.service'.
Authenticating as: Markus Persson (mcs)
Password: 
==== AUTHENTICATION COMPLETE ===
```

Do you have any more installations of Java on your server?
```bash
per@sson:/var/games/minecraft/profiles$ sudo update-alternatives --config java
[sudo] password for mcs: 
There is only one alternative in 
```

```bash
wget https://download.java.net/java/GA/jdk16.0.2/d4a915d82b4c4fbb9bde534da945d746/7/GPL/openjdk-16.0.2_linux-x64_bin.tar.gz

>sudo mkdir -p /usr/java/openjdk
>cd /usr/java/openjdk
>sudo cp /data/setups/openjdk-16_linux-x64_bin.tar.gz openjdk-16_linux-x64_bin.tar.gz
>sudo tar -xzvf openjdk-16_linux-x64_bin.tar.gz
```

Add these couple of lines tot the `/etc/profile` file:
```bash
per@sson:/usr/java/openjdk$ 
per@sson:/usr/java/openjdk$ sudo vim /etc/profile
# OpenJDK 16
JAVA_HOME=/usr/java/openjdk/jdk-16
PATH=$PATH:$HOME/bin:$JAVA_HOME/bin
export JAVA_HOME
export PATH
```


```bash
per@sson:/usr/java/openjdk$ 
per@sson:/usr/java/openjdk$ java -version
openjdk version "1.8.0_292"
OpenJDK Runtime Environment (build 1.8.0_292-8u292-b10-0ubuntu1~20.04-b10)
OpenJDK 64-Bit Server VM (build 25.292-b10, mixed mode)
per@sson:/usr/java/openjdk$ 
per@sson:/usr/java/openjdk$ sudo update-alternatives --install "/usr/bin/java" "java" "/usr/java/openjdk/jdk-16.0.2/bin/java" 1
per@sson:/usr/java/openjdk$ 
per@sson:/usr/java/openjdk$ sudo update-alternatives --install "/usr/bin/javac" "javac" "/usr/java/openjdk/jdk-16.0.2/bin/javac" 1
update-alternatives: using /usr/java/openjdk/jdk-16.0.2/bin/javac to provide /usr/bin/javac (javac) in auto mode
per@sson:/usr/java/openjdk$ java -version
openjdk version "1.8.0_292"
OpenJDK Runtime Environment (build 1.8.0_292-8u292-b10-0ubuntu1~20.04-b10)
OpenJDK 64-Bit Server VM (build 25.292-b10, mixed mode)
per@sson:/usr/java/openjdk$ 
per@sson:/usr/java/openjdk$ sudo update-alternatives --config java
There are 2 choices for the alternative java (providing /usr/bin/java).

  Selection    Path                                            Priority   Status
------------------------------------------------------------
* 0            /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java   1081      auto mode
  1            /usr/java/openjdk/jdk-16.0.2/bin/java            1         manual mode
  2            /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java   1081      manual mode

Press <enter> to keep the current choice[*], or type selection number: 1
update-alternatives: using /usr/java/openjdk/jdk-16.0.2/bin/java to provide /usr/bin/java (java) in manual mode
per@sson:/usr/java/openjdk$ 
per@sson:/usr/java/openjdk$ sudo update-alternatives --config java
There are 2 choices for the alternative java (providing /usr/bin/java).

  Selection    Path                                            Priority   Status
------------------------------------------------------------
  0            /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java   1081      auto mode
* 1            /usr/java/openjdk/jdk-16.0.2/bin/java            1         manual mode
  2            /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java   1081      manual mode

Press <enter> to keep the current choice[*], or type selection number: 

per@sson:/usr/java/openjdk$ java -version
openjdk version "16.0.2" 2021-07-20
OpenJDK Runtime Environment (build 16.0.2+7-67)
OpenJDK 64-Bit Server VM (build 16.0.2+7-67, mixed mode, sharing)
per@sson:/usr/java/openjdk$ 
per@sson:/usr/java/openjdk$ systemctl start mineos
```

### Screen
Your servers are attachable through screen (to view events):
```bash
screen -r
```

---

## Authors
Mr. Johnson

---

## Acknowledgments
* [https://minecraft.gamepedia.com/Server.properties](https://minecraft.gamepedia.com/Server.properties)
* [http://minecraft.codeemo.com/mineoswiki/index.php?title=MineOS-node_(apt-get)](http://minecraft.codeemo.com/mineoswiki/index.php?title=MineOS-node_(apt-get))
* [https://discourse.codeemo.com/t/permissions-ownership/3402/5](https://discourse.codeemo.com/t/permissions-ownership/3402/5)
* [https://www.server-world.info/en/note?os=Ubuntu_20.04&p=dhcp&f=2](https://www.server-world.info/en/note?os=Ubuntu_20.04&p=dhcp&f=2)
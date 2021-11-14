---
layout: default
title: How to install ZNC bouncer on Ubuntu
parent: Linux
nav_order: 21
---
# How to install ZNB bouncer on Ubuntu 
{: .no_toc }
ZNC is a bouncer for IRC. This is how I installed it on an Proxmox VM using Docker CE, Portainer (optional) and Watchtower (optional). I also used the ability through Nginx Reverse Proxy Manager to secure my cononection with an official certificate. 

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


### Set your time zone
```bash
read@arr:~$ date
Sat 11 Jan 21:22:53 GMT 2020
per@sson:~$ sudo dpkg-reconfigure tzdata

Current default time zone: 'Europe/Paris'
Local time is now:      Sat Jan 11 22:24:07 CET 2020.
Universal Time is now:  Sat Jan 11 21:24:07 UTC 2020.

read@arr:~$ $ date
Sat 11 Jan 22:24:16 CET 2020
```

### Disable IPv6
```bash
read@arr:~$ sudo nano /etc/default/grub
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
read@arr:~$ sudo update-grub
```

### Change NTP server
```bash
read@arr:~$ nano /etc/systemd/timesyncd.conf 
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
read@arr:~$ sudo systemctl restart systemd-timesyncd
```

### qemu-guest-agent
Install `qemu-guest-agent` for Proxmox VMs.
```bash
spider@man~$ sudo apt update
spider@man~$ sudo apt upgrade
spider@man~$ sudo apt install qemu-guest-agent
```
Shut down the VM. Enable Qemu Guest Agent in Proxmox and start the VM. 

---

## Docker
Installation of Docker:

* [https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04)

---

## Portainer
Install portainer to give an graphical overview for our Docker container. 

* [https://hub.docker.com/r/portainer/portainer-ce](https://hub.docker.com/r/portainer/portainer-ce)

```bash
spider@man:~$ sudo docker volume create portainer_data
portainer_data

sudo docker pull portainer/portainer-ce

sudo docker run -d -p 8000:8000 -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce
```

---

## ZNC
Installation of ZNC:

* [https://hub.docker.com/_/znc](https://hub.docker.com/_/znc)

```bash
spider@man:~$ docker pull znc
```

```bash
spider@man:~$ sudo docker volume create znc_data
znc_data
```

Create configuration file:
```bash
spider@man:~$ sudo docker run -it -v znc_data:/znc-data znc --makeconf
[ .. ] Checking for list of available modules...
[ ** ] 
[ ** ] -- Global settings --
[ ** ] 
[ ?? ] Listen on port (1025 to 65534): 1337
[ ?? ] Listen using SSL (yes/no) [no]: yes
[ ?? ] Listen using both IPv4 and IPv6 (yes/no) [yes]: no
[ .. ] Verifying the listener...
[ ** ] Unable to locate pem file: [/znc-data/znc.pem], creating it
[ .. ] Writing Pem file [/znc-data/znc.pem]...
[ ** ] Enabled global modules [webadmin]
[ ** ] 
[ ** ] -- Admin user settings --
[ ** ] 
[ ?? ] Username (alphanumeric): han
[ ?? ] Enter password: 
[ ?? ] Confirm password: 
[ ?? ] Nick [han]: 
[ ?? ] Alternate nick [han]: solo
[ ?? ] Ident [han]: 
[ ?? ] Real name (optional): J.F.K
[ ?? ] Bind host (optional): 
[ ** ] Enabled user modules [chansaver, controlpanel]
[ ** ] 
[ ?? ] Set up a network? (yes/no) [yes]: yes
[ ** ] 
[ ** ] -- Network settings --
[ ** ] 
[ ?? ] Name [freenode]: 
[ ?? ] Server host [chat.freenode.net]: 
[ ?? ] Server uses SSL? (yes/no) [yes]: 
[ ?? ] Server port (1 to 65535) [6697]: 
[ ?? ] Server password (probably empty): 
[ ?? ] Initial channels: linux
[ ** ] Enabled network modules [simple_away]
[ ** ] 
[ .. ] Writing config [/znc-data/configs/znc.conf]...
[ ** ] 
[ ** ] To connect to this ZNC you need to connect to it as your IRC server
[ ** ] using the port that you supplied.  You have to supply your login info
[ ** ] as the IRC server password like this: user/network:pass.
[ ** ] 
[ ** ] Try something like this in your IRC client...
[ ** ] /server <znc_server_ip> +1337 han:<pass>
[ ** ] 
[ ** ] To manage settings, users and networks, point your web browser to
[ ** ] https://<znc_server_ip>:1337/
[ ** ] 
[ ?? ] Launch ZNC now? (yes/no) [yes]: yes
[ .. ] Opening config [/znc-data/configs/znc.conf]...
[ .. ] Loading global module [webadmin]...
[ .. ] Binding to port [+1337] using ipv4...
[ ** ] Loading user [billard]
[ ** ] Loading network [freenode]
[ .. ] Loading network module [simple_away]...
[ >> ] [/opt/znc/lib64/znc/simple_away.so]
[ .. ] Adding 1 servers...
[ .. ] Loading user module [chansaver]...
[ .. ] Loading user module [controlpanel]...
[ ** ] Staying open for debugging [pid: 10]
[ ** ] ZNC 1.8.2 - https://znc.in
```
CTRL + C

### Run
```bash
sudo docker run -d -p 1337:1337 -v znc_data:/znc-data znc
```

### Configuration
* [https://address:1337](https://address:1337)

Username should be different from nickname. 

```bash 
/znc AddNetwork freenode
```

---

## Textual
### Server Properties
#### General
* Connection Name: 
* Server Address:
* Port: 
* Server Password: han's password

#### Identity
* Username: solo

---

## Watchtower 
To automatically update your ZNC container, you can use `watchtower`:
```
sudo docker run -d \
    --name watchtower \
    -v /var/run/docker.sock:/var/run/docker.sock \
    containrrr/watchtower \
    znc
```

---

## NPM
Generate a certificate through NPM. Download the certificate, e.g. irc.example.com

```bash
cat privkey1.pem > fullchain-with-privkey.pem
cat fullchain1.pem >> fullchain-with-privkey.pem

cp fullchain-with-privkey.pem znc.pem
chmod 600 znc.pem
```

Remember to do a NAT portforwarding rule to your ZNC server's ip address:port (1337) on your gateway, and maybe a host override in your network as well to allow connections using the certificate internally. 

---

## Authors
Mr. Johnson

---

## Acknowledgments
* [https://www.reddit.com/r/irc/comments/14p6it/how_do_i_remain_logged_into_a_server_when_idling/](https://www.reddit.com/r/irc/comments/14p6it/how_do_i_remain_logged_into_a_server_when_idling/)
* [https://www.letscloud.io/community/how-to-install-portainer](https://www.letscloud.io/community/how-to-install-portainer)
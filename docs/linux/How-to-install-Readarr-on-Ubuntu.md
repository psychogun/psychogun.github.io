---
layout: default
title: How to install Readarr on Ubuntu
parent: Linux
nav_order: 9
---
# How to install Readarr on Ubuntu 
{: .no_toc }
Readarrrrrrrrrrrrr.  I am using a container created by [https://hub.docker.com/r/hotio/readarr](hotio) for my Readarr setup. This setup utilizes SMB shares for storing eBooks, Deluge for downloading and Watchtower to automate updates of the `hotio/readarr `container. All visible through Portainer (optional).

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
* [https://hub.docker.com/r/hotio/readarr](https://hub.docker.com/r/hotio/readarr)
* Docker
* portainer-ce
* watchtower

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
```bash
read@arr:~$ sudo apt update
read@arr:~$ sudo apt upgrade
read@arr:~$ sudo apt install qemu-guest-agent
```
Shut down the VM. Enable Qemu Guest Agent in Proxmox and start the VM. 

---

## Prerequisites for Readarr
* SMB Shares
* Deluge SSL 
* Portainer

---

## SMB Shares
### Create mount point
I'll store the books on another server which will be shared to our Readarr installation through SMB. 

```bash
read@arr:~$  sudo apt install cifs-utils
```

Make a folder which we will use to mount:
```bash
read@arr:~$  cd /mnt
sudo mkdir SMB
cd SMB
sudo mkdir Readarr
sudo mkdir Torrents
```

### Add user group
```
read@arr:/etc/ssl/certs$ sudo groupadd -g 922 deluge
read@arr:/etc/ssl/certs$ groups read
read: read adm cdrom sudo dip plugdev lxd
read@arr:/etc/ssl/certs$ sudo usermod -a -G deluge readarr
read@arr:/etc/ssl/certs$ groups read
read : read adm cdrom sudo dip plugdev lxd deluge
```

### Automatic mount 
```bash
sudo nano /etc/fstab

#Torrents
//10.10.10.13/Torrents /mnt/SMB/Torrents cifs uid=1000,gid=1000,credentials=/root/.smbpasswd 0 0 

# Readarr
//10.10.10.13/Readarr /mnt/SMB/Readarr cifs uid=1000,gid=1000,credentials=/root/.smbpassword 0 0
```

Create credentials file:
```bash
read@arr:~$ sudo -i
[sudo] password for readarr: 

root@arr:~# cd ~
vim .smbpasswd

username=readarr
password=PASSWORD
root@arr:~$ sudo chmod 600 .smbpasswd
root@arr:~$ exit
```

View mounts:
```bash
read@arr:/mnt/SMB$ df -H
Filesystem                         Size  Used Avail Use% Mounted on
udev                               998M     0  998M   0% /dev
tmpfs                              188M  1.2M  187M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   22G  6.4G   14G  32% /
tmpfs                              563M     0  563M   0% /sys/fs/cgroup
tmpfs                              113M     0  113M   0% /run/user/1000
read@arr:/mnt/SMB$ 

```

Mount all:
```bash
sudo mount -a
```

View mounts:
```bash
read@arr:~$ df -H
Filesystem                         Size  Used Avail Use% Mounted on
udev                               998M     0  998M   0% /dev
tmpfs                              188M  1.2M  187M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   22G  6.4G   14G  32% /
tmpfs                              563M     0  563M   0% /sys/fs/cgroup
tmpfs                              113M     0  113M   0% /run/user/1000
/dev/sda2                          1.1G  207M  747M  22% /boot
//10.10.10.13/Torrents              0G    0G   47M 100% /mnt/SMB/Torrents
//10.10.10.13/Readarr             188G  1.1G  187G   1% /mnt/SMB/Readarr
```

### Manual mount SMB shares
```bash
read@arr:~$  sudo mount -t cifs -o username=readarr,uid=1000,gid=1000 //10.10.10.13/Readarr /mnt/SMB/Readarr

read@arr:~$ sudo mount -t cifs -o username=rradaer,uid=1000,gid=1000 //10.10.10.13/Torrents /mnt/SMB/Torrents
Password for rradaer@//10.10.10.13/Torrents: 
```


```json
    volumes:  
      - /mnt/SMB/Readarr:/mnt/Readarr
```

---

## Deluge
### Custom SSL 
View your custom certificate authority with `more`.
```bash
don@pablo:~/easy-rsa/pki$ more custom_ca.crt
```
Paste the information in a file called `custom_ca.crt` on our host: 
```bash
read@arr:~$ cd /etc/ssl/certs
read@arr:/etc/ssl/certs$ sudo nano custom_ca.crt
```

```json
    volumes:  
      - /etc/ssl/certs:/etc/ssl/certs:ro
```

First, copy your CA to  `/usr/local/share/ca-certificates``

```
sudo cp foo.crt /usr/local/share/ca-certificates/foo.crt

```
then, update CA store
```
sudo update-ca-certificates
```
That's all. You should get this output:
```bash
Updating certificates in /etc/ssl/certs... 1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d....
Adding debian:foo.pem
done.
done.
```

---

## Docker
We install the necessary packages to be able to install Docker:
```bash
read@arr:~$ sudo apt install apt-transport-https ca-certificates curl software-properties-common
```
We add the official Docker GPG key:
```bash
read@arr:~$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
OK
```
We activate Docker repository and update it:
```bash
read@arr:~$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
read@arr:~$ sudo apt update
```
We install the ***latest*** Docker version:
```bash
read@arr:~$ sudo apt install docker-ce
```

---

## Portainer
I use <kbd>Portainer</kbd> for managing Docker <kbd>containers</kbd>. Installing <kbd>Portainer</kbd> is very simple since it works in a Docker container, for this we will execute:
```bash
read@arr:~$ sudo docker volume create portainer_data
read@arr:~$ sudo docker run -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce
read@arr:~$ sudo docker ps
CONTAINER ID        IMAGE                    COMMAND             CREATED             STATUS              PORTS                              NAMES
83078d35b6a2        portainer/portainer-ce   "/portainer"        4 seconds ago       Up 2 seconds        8000/tcp, 0.0.0.0:9000->9000/tcp   keen_robinson
read@arr:~$ sudo docker update --restart unless-stopped 83078d35b6a2
```

### Admin user
Create an admin user through interface http://ip-address:90000, after which you have opted for `Connect local`.

Connect ***locally***. 

---

## Readarr
Install Readarr:
```bash
sudo docker volume create readarr_data
```

First time run to see if everything checks out OK:
```bash
sudo docker run -p 8787:8787 -e PUID=1000 -e PGID=1000 -e UMASK=002 -e TZ="Etc/UTC" -v readarr_data:/config -v /mnt/SMB/Torrents:/mnt/Torrents -v /etc/ssl/certs:/etc/ssl/certs -v /mnt/SMB/Readarr:/mnt/Readarr -v /usr/local/share/ca-certificates:/usr/local/share/ca-certificates --restart unless-stopped hotio/readarr:nightly 
```

Daemonize it (`-d`):
```bash
sudo docker run -d -p 8787:8787 -e PUID=1000 -e PGID=1000 -e UMASK=002 -e TZ="Etc/UTC" -v readarr_data:/config -v /mnt/SMB/Torrents:/mnt/Torrents -v /etc/ssl/certs:/etc/ssl/certs -v /mnt/SMB/Readarr:/mnt/Readarr -v /usr/local/share/ca-certificates:/usr/local/share/ca-certificates --restart unless-stopped hotio/readarr:nightly 
```

### Certificate
Have to do some more research here. 

General > Security > Certificate Validation * DISABLED *

### Remote Path Mappings
* Host: 10.10.10.13
* Remote Path: mnt/Torrents/Finished/
* Local Path: /mnt/SMB/Torrents/Finished/

---

## Watchtower
Auto update specific container `readarr`:
```bash
read@arr:~$ sudo docker pull containrrr/watchtower
Using default tag: latest
latest: Pulling from containrrr/watchtower
9b3d60226310: Pull complete 
b20cee7b2b25: Pull complete 
78104bd4b0bf: Pull complete 
Digest: sha256:d0331edc5b1c5bbf18a92fc27c50f32e8bb894cf67a06cbd33d04eb40d5c8cc2
Status: Downloaded newer image for containrrr/watchtower:latest
docker.io/containrrr/watchtower:latest
```

```bash
read@arr:~$ sudo docker run -d --restart=always --name watchtower -v /var/run/docker.sock:/var/run/docker.sock containrrr/watchtower readarr watchtower --stop-timeout 300s
```

---

## Fault finding
### CLI
Log in to your container through CLI:
```
read@arr:~$ sudo docker ps
CONTAINER ID   IMAGE                   COMMAND   CREATED         STATUS         PORTS                                       NAMES
8fc303220eeb   hotio/readarr:nightly   "/init"   5 minutes ago   Up 5 minutes   0.0.0.0:8787->8787/tcp, :::8787->8787/tcp   practical_colden

sudo docker exec -it 8fc303220eeb /bin/bash
```

### Make portainer start at boot
Go to Containers, select the portainer container and under `Container details`, select `Restart policies` and use `Unless stopped` and press `Update`. 

---

## Authors
Mr. Johnson

---

## Acknowledgments 

* [https://askubuntu.com/questions/73287/how-do-i-install-a-root-certificate](https://askubuntu.com/questions/73287/how-do-i-install-a-root-certificate)
* [https://www.reddit.com/r/sonarr/comments/jpkf5c/deluge_error_in_sonarr_after_upgrade_to_version/](https://www.reddit.com/r/sonarr/comments/jpkf5c/deluge_error_in_sonarr_after_upgrade_to_version/)
* [https://phoenixnap.com/kb/update-docker-image-container](https://phoenixnap.com/kb/update-docker-image-container)
* [https://github.com/Readarr/Readarr/wiki/Docker](https://github.com/Readarr/Readarr/wiki/Docker)
* [https://docs.docker.com/config/containers/start-containers-automatically/](https://docs.docker.com/config/containers/start-containers-automatically/)
* [https://linoxide.com/how-tos/howto-mount-smb-filesystem-using-etcfstab/](https://linoxide.com/how-tos/howto-mount-smb-filesystem-using-etcfstab/)
* [https://askubuntu.com/questions/1245825/cant-write-to-smbclient-mounted-network-disk-without-sudo](https://askubuntu.com/questions/1245825/cant-write-to-smbclient-mounted-network-disk-without-sudo)
* [https://www.caretech.io/2019/06/06/how-to-install-a-custom-certificate-authority-for-the-linux-command-line/](https://www.caretech.io/2019/06/06/how-to-install-a-custom-certificate-authority-for-the-linux-command-line/)
* [https://documentation.storj.io/setup/cli/storage-node](https://documentation.storj.io/setup/cli/storage-node)
* [https://github.com/containrrr/watchtower](https://github.com/containrrr/watchtower)
* [http://timlehr.com/auto-mount-samba-cifs-shares-via-fstab-on-linux/](http://timlehr.com/auto-mount-samba-cifs-shares-via-fstab-on-linux/)

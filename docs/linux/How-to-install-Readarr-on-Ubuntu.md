---
layout: default
title: How to install Readarr on Ubuntu
parent: Linux
nav_order: 12
---
# How to install Readarr on Ubuntu 
{: .no_toc }
Readarrrrrrrrrrrrr. 

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}
---
## Getting started


## Prerequisites
* Proxmox 6.1
* Ubuntu 20.04
* [https://hub.docker.com/r/hotio/readarr](https://hub.docker.com/r/hotio/readarr)

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

## Portainer
### Prerequisites
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

### Installation
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


### Make portainer start at boot
Go to Containers, select the portainer container and under `Container details`, select `Restart policies` and use `Unless stopped` and press `Update`. 

### watchtower
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
read@arr:/etc/ssl/certs$ sudo groupadd -g 922 deluge

read@arr:/etc/ssl/certs$ groups read
read: read adm cdrom sudo dip plugdev lxd

read@arr:/etc/ssl/certs$ sudo usermod -a -G deluge readarr

read@arr:/etc/ssl/certs$ groups read
read : read adm cdrom sudo dip plugdev lxd deluge


### Automatic mount 
```bash
sudo nano /etc/fstab

#Torrents
//10.10.10.13/Torrents /mnt/SMB/Torrents cifs uid=1000,gid=1000,credentials=/root/.smbpasswd 0 0 

# Readarr
//10.10.10.13/Readarr /mnt/SMB/Readarr cifs uid=1000,gid=1000,credentials=/root/.smbpassword 0 0
```

```bash
read@arr:~$ sudo -i
[sudo] password for readarr: 

root@h37breadarr:~# cd ~
nano .smbpasswd

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
//192.168.5.13/Torrents              0G    0G   47M 100% /mnt/SMB/Torrents
//192.168.5.13/Readarr             188G  1.1G  187G   1% /mnt/SMB/Readarr
```

### Manual mount SMB shares
```bash
read@arr:~$  sudo mount -t cifs -o username=readarr,uid=1000,gid=1000 //192.168.5.13/Readarr /mnt/SMB/Readarr

read@arr:~$ sudo mount -t cifs -o username=rradaer,uid=1000,gid=1000 //192.168.5.13/Torrents /mnt/SMB/Torrents
Password for rradaer@//192.168.5.13/Torrents: 
```


```json
    volumes:  
      - /mnt/SMB/Readarr:/mnt/Readarr
```


cd /mnt/SMB
sudo mkdir Torrents


sudo mount -t cifs -o username=rradaer,uid=1000,gid=1000 //192.168.5.13/Torrents /mnt/SMB/Torrents
Password for rradaer@//192.168.5.13/Torrents: 

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

### Remote Path Mappings
Remote Path Mappings
Host
Remote Path
Local Path
192.168.243.111
/mnt/Torrents/Finished/
/mnt/SMB/Torrents/Finished/




## Authors
Mr. Johnson


## Acknowledgments 
* [https://github.com/Readarr/Readarr/wiki/Docker](https://github.com/Readarr/Readarr/wiki/Docker)
* [https://docs.docker.com/config/containers/start-containers-automatically/](https://docs.docker.com/config/containers/start-containers-automatically/)
* [https://linoxide.com/how-tos/howto-mount-smb-filesystem-using-etcfstab/](https://linoxide.com/how-tos/howto-mount-smb-filesystem-using-etcfstab/)
* [https://askubuntu.com/questions/1245825/cant-write-to-smbclient-mounted-network-disk-without-sudo](https://askubuntu.com/questions/1245825/cant-write-to-smbclient-mounted-network-disk-without-sudo)
* [https://www.caretech.io/2019/06/06/how-to-install-a-custom-certificate-authority-for-the-linux-command-line/](https://www.caretech.io/2019/06/06/how-to-install-a-custom-certificate-authority-for-the-linux-command-line/)
* [https://documentation.storj.io/setup/cli/storage-node](https://documentation.storj.io/setup/cli/storage-node)
* [https://github.com/containrrr/watchtower](https://github.com/containrrr/watchtower)
* [http://timlehr.com/auto-mount-samba-cifs-shares-via-fstab-on-linux/](http://timlehr.com/auto-mount-samba-cifs-shares-via-fstab-on-linux/)

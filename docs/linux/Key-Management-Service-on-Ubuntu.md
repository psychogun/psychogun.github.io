---
layout: default
title: Key Management Service on Ubuntu
parent: Linux
nav_order: 10
---
# Key Management Service on Ubuntu
{: .no_toc }
This is how I used `kms-server` on an Ubuntu 

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}
---
## Getting started


## Prerequisites
* Proxmox Virtual Environment 6.1-5
* Ubuntu 18.04.3 Server


## Update
```bash
sudo apt-get update
sudo apt-get upgrade
```

## Install qemu-guest-agent
The qemu-guest-agent is a helper daemon, which is installed in the guest. It is used to exchange information between the host and guest, and to execute command in the guest.

In Proxmox VE, the qemu-guest-agent is used for mainly two things:

To properly shutdown the guest, instead of relying on ACPI commands or windows policies
To freeze the guest file system when making a backup (on windows, use the volume shadow copy service VSS).

```bash
administrator@kms:~$ sudo apt-get install qemu-guest-agent
administrator@kms:~$ sudo shutdown now
```
In Proxmox, go to Options and Enable by selecting `Use QEMU Guest Agent`. Start your VM again. 

## Installing the KMS server
The KMS server is available as binaries or source code that needs to be compiled. Let us use the source code.

``bash
administrator@kms:~$ sudo apt-get install git gcc make
```
`git clone` the KMS server source code:
```bash
administrator@kms:~$ git clone https://github.com/Wind4/vlmcsd
````
Compile the source code:
```bash
administrator@kms:~$ cd vlmcsd
administrator@kms:~/vlmcsd$ make
```
### Start the KMS daemon
Start the KMS daemon with `./vlmcsd`:
```bash
administrator@kms:~/vlmcsd$ cd bin/
administrator@kms:~/vlmcsd/bin$ ./vlmcsd 
```
Using `netstat` you'll see that `vlmcd` is listening on 1688:
```bash
administrator@kms:~/vlmcsd/bin$ netstat -a
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 0.0.0.0:1688            0.0.0.0:*               LISTEN  
```
### Test with KMS client
Test the KMS server by running the KMS client:
```bash
administrator@kms:~/vlmcsd/bin$ ./vlmcs
Connecting to 127.0.0.1:1688 ... successful
Sending activation request (KMS V6) 1 of 1  -> 1111-12201-369-033755-01-5079-11213.0010-0032021 (4A1DD49C13BB0079)
```
## Configuring the KMS daemon to run at boot
```bash
administrator@kms:~/vlmcsd/bin$ sudo nano /etc/systemd/system/kms@administrator.service
[Unit]
Description=KMS
After=network-online.target

[Service]
Type=forking
WorkingDirectory=/home/administrator/vlmcsd/bin
User=administrator
ExecStart=/home/administrator/vlmcsd/bin/vlmcsd

[Install]
WantedBy=multi-user.target
```

You need to reload systemd to make the daemon aware of the new configuration.
```bash
administrator@kms:~$ sudo systemctl --system daemon-reload
```
To have the KMS daemon start automatically at boot, enable the service.
```bash
administrator@kms:/etc/systemd/system$ sudo systemctl enable kms@administrator.service
Created symlink /etc/systemd/system/multi-user.target.wants/kms@administrator.service → /etc/systemd/system/kms@administrator.service.
```
To disable the automatic start, use this command.
```bash
administrator@kms:/etc/systemd/system$ sudo systemctl disable kms@administrator.service 
Removed /etc/systemd/system/multi-user.target.wants/kms@administrator.service.
```

To start the KMS daemon now, use this command.
```bash
administrator@kms:/etc/systemd/system$ sudo systemctl start kms@administrator.service 
```

You can also substitute the `start` above with `stop` to stop Home Assistant, `restart` to restart Home Assistant, and `status` to see a brief status report as seen below.
```bash
administrator@kms:/etc/systemd/system$ sudo systemctl status kms@administrator.service 
● kms@administrator.service - KMS
   Loaded: loaded (/etc/systemd/system/kms@administrator.service; enabled; vendor preset: enabled)
   Active: inactive (dead) since Mon 2020-02-03 14:31:21 UTC; 1s ago
  Process: 6491 ExecStart=/home/administrator/vlmcsd/bin/vlmcsd (code=exited, status=0/SUCCESS)
 Main PID: 6491 (code=exited, status=0/SUCCESS)

Feb 03 14:31:21 kms systemd[1]: Started KMS.
```

```bash
administrator@kms:/etc/systemd/system$ ps -ef | grep vlmcsd
adminis+  6363  5391  0 14:29 pts/0    00:00:00 grep --color=auto vlmcsd
```

## Windows configuration
Open up `cmd.exe` as Administrator:
```cmd
C:\Windows\system32\> slmgr /skms YOUR_IP_OR_HOSTNAME
C:\Windows\system32\> slmgr /ato
```

slmgr /upk
slmgr /ipk XXXXX-XXXXX-XXXXX-XXXXX-XXXXX
slmgr /skms YOUR_IP_OR_HOSTNAME
slmgr /ato


CD \Program Files\Microsoft Office\Office16 OR CD \Program Files (x86)\Microsoft Office\Office16
cscript ospp.vbs /sethst:YOUR_IP_OR_HOSTNAME
cscript ospp.vbs /inpkey:xxxxx-xxxxx-xxxxx-xxxxx-xxxxx
cscript ospp.vbs /act
cscript ospp.vbs /dstatusall



## Authors
Mr. Johnson


## Acknowledgments
* [https://github.com/kebe7jun/linux-kms-server](https://github.com/kebe7jun/linux-kms-server)
* [https://ryanrudolfoba.com/blog/2019-06-12-hosting-my-own-kms-server-and-creating-bind-dns-records-for-streamlined-windows-and-office-activation/](https://ryanrudolfoba.com/blog/2019-06-12-hosting-my-own-kms-server-and-creating-bind-dns-records-for-streamlined-windows-and-office-activation/)
* [https://pve.proxmox.com/wiki/Qemu-guest-agent](https://pve.proxmox.com/wiki/Qemu-guest-agent)
---
layout: default
title: How to install Jackett
parent: Linux
nav_order: 8
---
# How to install Jackett on Ubuntu
{: .no_toc }
Jackettttttttt.

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
* [https://github.com/Jackett/Jackett](https://github.com/Jackett/Jackett)

### Prerequisites
* Proxmox Virtual Environment 6.1-5
* Ubuntu 18.04.3 Server


### Update
```bash
sudo apt-get update
sudo apt-get upgrade
```

### Install qemu-guest-agent
The qemu-guest-agent is a helper daemon, which is installed in the guest. It is used to exchange information between the host and guest, and to execute command in the guest.

In Proxmox VE, the qemu-guest-agent is used for mainly two things:

To properly shutdown the guest, instead of relying on ACPI commands or windows policies
To freeze the guest file system when making a backup (on windows, use the volume shadow copy service VSS).

```bash
jack@ett:~$ sudo apt-get install qemu-guest-agent
jack@ett::~$ sudo shutdown now
```
In Proxmox, go to Options and Enable by selecting `Use QEMU Guest Agent`. Start your VM again. 

---

## .NET
[https://github.com/dotnet/core/blob/master/Documentation/linux-prereqs.md](https://github.com/dotnet/core/blob/master/Documentation/linux-prereqs.md)
```bash
jack@ett:/tmp$ sudo apt-get install libcurl4-openssl-dev bzip2 mono-devel
```

---

## Install Jackett
Go to [https://github.com/Jackett/Jackett/releases](https://github.com/Jackett/Jackett/releases), download the latest `Jackett.Binaries.Mono.LinuxAMD64.tar.gz` (copy link):
```bash
jack@ett::~$ cd /tmp/
jack@ett::/tmp$ wget https://github.com/Jackett/Jackett/releases/download/v0.14.365/Jackett.Binaries.LinuxAMDx64.tar.gz
```
Unpack the Jackett release:
```bash
jack@ett::/tmp$ tar -xvf Jackett.Binaries.LinuxAMDx64.tar.gz 
```
Make the Jackett installation folder:
```bash
jack@ett::/tmp$ sudo mkdir /opt/jackett
```
Move the unzipped Jackett installation:
```bash
jack@ett::/tmp$ sudo mv Jackett/* /opt/jackett/
```
Change ownership of `/opt/jackett` to your main user:
```bash
jack@ett::/tmp$ sudo chown -R jack:jack /opt/jackett/
```
Test running Jackett which runs on port 9117 http://ip.address:9117
```
jack@ett::/tmp$ cd /opt/jackett/
jack@ett::/opt/jackett$ ./jackett
```

Runs OK? Then you can opt in to start Jackett at boot. Let us execute service systemd script to configure this:

```bash
jack@ett::/opt/jackett$ ./jackett sudo ./install_service_systemd.sh
jack@ett::/opt/jackett$ sudo systemctl status jackett.service
jack@ett::/opt/jackett$ sudo systemctl enable jackett.service
```


---

## Fault finding

```bash
root@jackett:~# curl 127.0.0.1:9117
curl: (7) Failed to connect to 127.0.0.1 port 9117: Connection refused
```

This is what I have found out so far:
```bash
root@ett:/home/jackett/_jackett# systemctl status jackett
* jackett.service - Jackett Daemon
     Loaded: loaded (/etc/systemd/system/jackett.service; enabled; vendor preset: enabled)
     Active: activating (auto-restart) since Tue 2021-05-18 15:01:43 CEST; 4s ago
    Process: 7994 ExecStart=/bin/sh /opt/jackett/jackett_launcher.sh (code=exited, status=0/SUCCESS)
   Main PID: 7994 (code=exited, status=0/SUCCESS)

May 18 15:01:43 ett systemd[1]: jackett.service: Succeeded.
```

But it is not listening on any interfaces:
```bash
root@ett:/home/jackett/_jackett# ss -tulp
Netid State   Recv-Q  Send-Q       Local Address:Port     Peer Address:Port Process                                    
udp   UNCONN  0       0            127.0.0.53%lo:domain        0.0.0.0:*     users:(("systemd-resolve",pid=106,fd=12)) 
udp   UNCONN  0       0        192.168.33.12%eth0:bootpc       0.0.0.0:*     users:(("systemd-network",pid=89,fd=18))  
tcp   LISTEN  0       4096         127.0.0.53%lo:domain        0.0.0.0:*     users:(("systemd-resolve",pid=106,fd=13)) 
tcp   LISTEN  0       128                0.0.0.0:ssh           0.0.0.0:*     users:(("sshd",pid=139,fd=3))             
tcp   LISTEN  0       100              127.0.0.1:smtp          0.0.0.0:*     users:(("master",pid=295,fd=13))          
tcp   LISTEN  0       128                   [::]:ssh              [::]:*     users:(("sshd",pid=139,fd=4))             
tcp   LISTEN  0       100                  [::1]:smtp             [::]:*     users:(("master",pid=295,fd=14)) 
```

Do not know what has happened, it was just down (probably a corrupted install)
```bash
root@ett:/home/jackett/.config/Jackett# tail updater.txt.20210425.00000.txt 
2021-04-25 05:16:19.2620 Info Copied libSystem.Native.so 
2021-04-25 05:16:19.2635 Info Attempting to copy libSystem.Net.Security.Native.a from source: /tmp/JackettUpdate-v0.17.938-637549245756703242/Jackett/libSystem.Net.Security.Native.a to destination: /opt/jackett/libSystem.Net.Security.Native.a 
2021-04-25 05:16:19.2698 Info Copied libSystem.Net.Security.Native.a 
2021-04-25 05:16:19.2698 Info Attempting to copy libSystem.Net.Security.Native.so from source: /tmp/JackettUpdate-v0.17.938-637549245756703242/Jackett/libSystem.Net.Security.Native.so to destination: /opt/jackett/libSystem.Net.Security.Native.so 
2021-04-25 05:16:19.2698 Info Copied libSystem.Net.Security.Native.so 
2021-04-25 05:16:19.2698 Info Attempting to copy libSystem.Security.Cryptography.Native.OpenSsl.a from source: /tmp/JackettUpdate-v0.17.938-637549245756703242/Jackett/libSystem.Security.Cryptography.Native.OpenSsl.a to destination: /opt/jackett/libSystem.Security.Cryptography.Native.OpenSsl.a 
2021-04-25 05:16:19.2723 Info Copied libSystem.Security.Cryptography.Native.OpenSsl.a 
2021-04-25 05:16:19.2723 Info Attempting to copy libSystem.Security.Cryptography.Native.OpenSsl.so from source: /tmp/JackettUpdate-v0.17.938-637549245756703242/Jackett/libSystem.Security.Cryptography.Native.OpenSsl.so to destination: /opt/jackett/libSystem.Security.Cryptography.Native.OpenSsl.so 
2021-04-25 05:16:19.2723 Info Copied libSystem.Security.Cryptography.Native.OpenSsl.so 
2021-04-25 05:16:19.2754 Info Attempting to copy LICENSE from source: /tmp/JackettUpdate-v0.17.938-637549245756703242/Jackett/LICENSE to destination: /opt/jackett/LICENSE 
```

Think I'll just try to backup my settings [https://github.com/Jackett/Jackett/issues/2576](https://github.com/Jackett/Jackett/issues/2576) and throw in a new install. ..

---

## Authors
Mr. Johnson

---

## Acknowledgments
* [https://varhowto.com/install-jackett-ubuntu-20-04/#Autostart_Jackett_with_systemd](https://varhowto.com/install-jackett-ubuntu-20-04/#Autostart_Jackett_with_systemd)
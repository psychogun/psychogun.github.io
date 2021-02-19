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
https://github.com/dotnet/core/blob/master/Documentation/linux-prereqs.md
```bash
sparrow@h37bjackett01:/tmp$ sudo apt-get install libcurl4-openssl-dev bzip2 mono-devel
```

---

## Install Jackett
Go to https://github.com/Jackett/Jackett/releases, download the latest `Jackett.Binaries.Mono.LinuxAMD64.tar.gz` (copy link):
```bash
jack@ett::~$ cd /tmp/
jack@ett::/tmp$ https://github.com/Jackett/Jackett/releases/download/v0.14.365/Jackett.Binaries.LinuxAMDx64.tar.gz
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

---

## Authors
Mr. Johnson
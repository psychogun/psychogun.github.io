---
layout: default
title: Installation of UniFi Video on Ubuntu 
parent: Linux
nav_order: 20
---
# Installation of UniFi Video on Ubuntu 
{: .no_toc }
This is how I installed `unifi-video` on an Ubuntu 20.04 server, to use with my Home Assistant installation. 

Even though Unifi-Video is deprecated; [https://help.ui.com/hc/en-us/articles/360057458834-Accessing-UniFi-Video-after-End-of-Support](https://help.ui.com/hc/en-us/articles/360057458834-Accessing-UniFi-Video-after-End-of-Support) - you are still able to install it. It will not receive any security updates or further development, so it might be not that wise to expose this instance fully on the world wide web, but hide it behind a VPN if you are connecting externally.  

The latest version I've found of `unifi-video` was ~~3.10.11~~ 3.10.13.

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

It was a  real pain in the butt to find the latest version of the `unifi-video` software, as it has been removed from the download site [https://www.ui.com/download/unifi-video/](https://www.ui.com/download/unifi-video/) / [https://www.reddit.com/r/Ubiquiti/comments/l94er8/does_anyone_know_where_i_can_download_unifi_video/](https://www.reddit.com/r/Ubiquiti/comments/l94er8/does_anyone_know_where_i_can_download_unifi_video/) 

 Ubiquiti has stopped developing UniFi-Video products, and people are forced to use UniFi-Protect instead which works only on dedicated hardware from Ubiquiti. A shame, really - as the comments do show [https://community.ui.com/questions/UniFi-Video-Products-End-of-Life-Announcement/dc529d39-0e58-43cc-96f0-8f0eed0d002c](https://community.ui.com/questions/UniFi-Video-Products-End-of-Life-Announcement/dc529d39-0e58-43cc-96f0-8f0eed0d002c).

However, I've found some downloads which should be appropriate for our manual installation:

* [https://dl.ui.com/firmwares/ufv/v3.10.11/unifi-video.Ubuntu18.04_amd64.v3.10.11.deb](https://dl.ui.com/firmwares/ufv/v3.10.11/unifi-video.Ubuntu18.04_amd64.v3.10.11.deb)
* [https://dl.ubnt.com/firmwares/ufv/v3.10.13/unifi-video.Debian7_amd64.v3.10.13.deb](https://dl.ubnt.com/firmwares/ufv/v3.10.13/unifi-video.Debian7_amd64.v3.10.13.deb)


Let's try to install `unifi-video.Ubuntu18.04_amd64.v3.10.11.deb` on this Ubuntu 20.04 installation of ours.

---

## Prerequisites
As always, I am using Proxmox. No further explanation here - for convenience, remember to install `qemu-guest-agent` (`sudo apt install qemu-guest-agent`). 

Disable IPv6, as we do not want `unifi-video` to bind to this address:
```bash
nvr@nvr:~$ sudo nano /etc/default/grub
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
nvr@nvr:~$ sudo update-grub
```

Install dependencies:
```bash
nvr@nvr:~$ sudo apt-get install mongodb mongodb-server openjdk-8-jre-headless jsvc
```
As we do not want to later on update our `openjdk` installation to a newer version than 8, do: 
```bash
nvr@nvr:/tmp$ sudo apt-mark hold openjdk-11-*
```

And, as I found out writing this, the installed Java version is too new for this old `unifi-video` installation. We will have to downgrade Java. Thanks to this post, [https://community.ui.com/questions/unifi-video-wont-start-anymore-FIX-INSIDE/297dbfc0-7e04-4a50-92b8-dab4acf50a03i](https://community.ui.com/questions/unifi-video-wont-start-anymore-FIX-INSIDE/297dbfc0-7e04-4a50-92b8-dab4acf50a03), it is fairly easy.

Download `jre1.8.0_271` from this direct link:
```bash
nvr@nvr:~$ sudo su
root@nvr:/tmp# mkdir /usr/local/java
root@nvr:/tmp# cd /usr/local/java
root@nvr:/usr/local/java# wget https://javadl.oracle.com/webapps/download/AutoDL?BundleId=243727_61ae65e088624f5aaa0b1d2d801acb16
root@nvr:/usr/local/java# tar zxvf AutoDL\?BundleId\=243727_61ae65e088624f5aaa0b1d2d801acb16 
```
You should now have a file called `jre1.8.0_271` in your `/usr/local/java` directory. We can remove the downloaded file with command `rm AutoDL\?BundleId\=243727_61ae65e088624f5aaa0b1d2d801acb16`.

Let us continue:
```bash
root@nvr:/usr/local/java# update-alternatives --install "/usr/bin/java" "java" "/usr/local/java/jre1.8.0_271/bin/java" 1
root@nvr:/usr/local/java#  update-alternatives --config java
There are 2 choices for the alternative java (providing /usr/bin/java).

  Selection    Path                                            Priority   Status
------------------------------------------------------------
* 0            /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java   1081      auto mode
  1            /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java   1081      manual mode
  2            /usr/local/java/jre1.8.0_271/bin/java            1         manual mode

Press <enter> to keep the current choice[*], or type selection number: 2
update-alternatives: using /usr/local/java/jre1.8.0_271/bin/java to provide /usr/bin/java (java) in manual mode
```

Update the environment:
```bash
root@nvr:/usr/local/java# echo "JAVA_HOME=/usr/local/java/jre1.8.0_271" | tee -a /etc/default/unifi
JAVA_HOME=/usr/local/java/jre1.8.0_271
```
Issue a `reboot now` / or `shutdown now` to enable Qemu Guest Agent in the Proxmox virtual host before starting it again.  

---

## Install unifi-video

```bash
nvr@nvr:~$  cd /tmp
nvr@nvr:/tmp$ wget https://dl.ui.com/firmwares/ufv/v3.10.11/unifi-video.Ubuntu18.04_amd64.v3.10.11.deb
nvr@nvr:/tmp$ sudo dpkg -i unifi-video.Ubuntu18.04_amd64.v3.10.11.deb
```

Check status
```bash
nvr@nvr:~$ sudo systemctl status unifi-video
```

If it hasn't started, try to start it:
```bash
nvr@nvr:~$ sudo systemctl start unifi-video
```

Now you can access your `unifi-video` installation at port `:7080 `in your webbrowser (`http`). After the initial configuration, all subsequent traffic should be used using `https` and port `:7443`, with the self-signed certificate from UniFi-Video. 

I had to use Google Chrome on this part, as Safari on my Mac did not work (everywhere I clicked, I was prompted to upload a file - I could not even give the installation a name). This software is old..


### Upgrade unfi-video
I did not check this post [https://community.ui.com/releases/UniFi-Video-3-10-13/7cca7ae9-f4ff-4844-a7c4-b8163bb81f21](https://community.ui.com/releases/UniFi-Video-3-10-13/7cca7ae9-f4ff-4844-a7c4-b8163bb81f21) thouroughly, as on the very bottom it had listed a newer version of `unifi-video` (Download Links). 

So let us upgrade our current installation:
```bash
nvr@nvr:~$ cd /tmp
nvr@nvr:/tmp$ wget https://dl.ubnt.com/firmwares/ufv/v3.10.13/unifi-video.Ubuntu18.04_amd64.v3.10.13.deb
nvr@nvr:/tmp$ sudo service unifi-video stop
nvr@nvr:/tmp$ sudo su
root@nvr:/tmp# /usr/lib/unifi-video/bin/ubnt.updater /tmp/unifi-video.Ubuntu18.04_amd64.v3.10.13.deb 
root@nvr:/tmp# service unifi-video start
```
---

## Recording disk
Datacenter > 102 (nvr) > Hardware > Add > Hard Disk

I've chosen to add a 200GB disk.

Reboot the VM. 

Use `sudo cfdisk /dev/sdb` to create a partition and set the type to Linux File System. Initially, set label to GPT.

Then use `sudo mkfs.ext4 /dev/sdb1` to reformat to ext4.

To check which file system you have, use `sudo file -sL /dev/sdb1`. It should state `ext4` somewhere.

To find out which UUID the disk has, use `blkid` - we'll use this output in a bit.
```bash
nvr@nvr:~$ sudo blkid
/dev/sdb1: UUID="85254ce0-4151-4232-9cd0-f7cf7f3c107e" TYPE="ext4" PARTUUID="03532 (...)
```


I want to mount this disk to `/mnt`, you can do wherever. First, let us create a mount point.
```bash
nvr@nvr:~$ cd /mnt
sudo unifi-video-recordings
```

Make the mount permanent by adding a entry in `/etc/fstab`;
```bash
nvr@nvr:~$ sudo vim /etc/fstab
# UniFi-Video recordings
/dev/disk/by-uuid/85245ce0-4151-4232-9cd0-f7cf7f3c107e /mnt/unifi-video-recordings ext4 defaults 0 1
```

Let's mount it by using `mount -a`. If no errors, success - you can also verify by issuing `df -hT` to see where the disk is mounted.

Create a folder in this mount point and give the user which executes `unifi-video` permissions on this folder.

I like to name the folder the same as the site name, e.g. "paris".
```bash
nvr@nvr:~$ sudo su
cd /mnt/unifi-video-recordings
mkdir paris
```

Using `more /etc/passwd | grep unifi` I take a wild guess and assume that the user `unifi-video` is the appropriate user to give permissions to this folder to:
```bash
chown -R unifi-video:unifi-video paris
```
(This was also confirmed by checking out the permissions on Ubiquitis default folder, `ls -alh /usr/lib/unifi-video/data/videos`)

Anyways, let's make sure noone can snoop around in this folder:
```bash
chmod -R 750 paris
```

Allright. Go to UniFi video in your web browser. Down on the left, click SETTINGS. 

* Recording path: /mnt/unifi-video-recordings/paris

Click Save.

Verify that it is working by setting a camera to always record. I am recording only motion, alter to suite your needs.

---

## Fault finding
### Logs
Check the logs;
```bash
nvr@nvr:~$ more /var/log/unifi-video/
```

### Update
`sudo apt update ; sudo apt upgrade -y`

---

## Authors
Mr. Johnson

---


## Acknowledgments
* [https://community.ui.com/questions/How-to-install-Unifi-Video-on-Ubuntu-18-04-Now-Supported/6dbb2c6b-af93-4150-9659-4fa0a72ca847](https://community.ui.com/questions/How-to-install-Unifi-Video-on-Ubuntu-18-04-Now-Supported/6dbb2c6b-af93-4150-9659-4fa0a72ca847)
* [https://help.ui.com/hc/en-us/articles/221314008-UniFi-Video-How-to-Utilize-RTSP-Directly-From-the-Camera](https://help.ui.com/hc/en-us/articles/221314008-UniFi-Video-How-to-Utilize-RTSP-Directly-From-the-Camera)
* [https://community.ui.com/releases/UniFi-Video-3-10-13/7cca7ae9-f4ff-4844-a7c4-b8163bb81f21](https://community.ui.com/releases/UniFi-Video-3-10-13/7cca7ae9-f4ff-4844-a7c4-b8163bb81f21)
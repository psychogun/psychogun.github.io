---
layout: default
title: Installation and configuration of Home Assistant
parent: Linux
nav_order: 3
---
*WORK IN PROGRESS as of 2019-08-30*
## Installation and configuration of Home Assistant
{: .no_toc }
This is how I used Home Assistant to send me events through Apple's Home app, with the help of an Ubuntu installation allocated on a VM in ESXi 6.7 with USB devices passed through the host. 
I have some lights, a smartlock, some devices to measure temperature and, hm, what else, a motion detector that triggers the light if it is dark. Read on. ..

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}
---

## Getting started
This is my guide. There are many like it, but this one is mine. My guide is my best friend. It is my life. 
I must master it as I must master my life. Without me, my guide is useless. Without my guide I am useless. I must trigger my guide true. 

### Prerequsites
Some basic knowledge of UNIX commands (and using SSH).

* ESXi 6.7
* Ubuntu 18.04
* Home Assistant
* Deconz Docker

* Aeotec ZW090 Z-Stick Gen5 EU
* Aeotec ZW100 MultiSensor 6
* FIBARO System FGBS001 Universal Binary Sensor
* FIBARO System FGWPE/F Wall plug Gen5
* ID Lock AS ID Lock 150
* Philips Hue light
* IKEA TRADFRI light
* Apple TV 4th gen

## Allocating a VM in ESXi 6.7
Log in to your ESXi in a web-browser. Press `Virtual Machines` and then `Create / Register VM`.

***Select creation type***:

Create a new virtual machine > Next

***Select a name and guest OS***

Name: goodgold
Compatibility: ESXi 6.7 Virtual Machine
Guest OS family: Linux
Guest OS version: Ubuntu Linux (64-bit)

***Select storage***

Standard
Select a fitting datastore for the virtual machine's configuration files and all of it's virtual disks > Next

***Customize settings***

I recommend 20GB for harddrive size, rest default, that is OK for now > Next

***Ready to complete***

Press Finish and you have created the VM which we are installing Ubuntu 18.04 on.

Right click on your newly created Virtual machine. Select `Edit settings`. Scroll down to CD/DVD Drive 1 and select `Datastore ISO file`. Navigate to your `ubuntu-18.04.1-live-server-amd64.iso` file`.  Select > Save. 

Power on the Virtual machine and follow this guide for an excellent guide to installing ubuntu server: [https://tutorials.ubuntu.com/tutorial/tutorial-install-ubuntu-server-1604](https://tutorials.ubuntu.com/tutorial/tutorial-install-ubuntu-server-1604#0) 
(even though the link is for 1604, almost the same applies for 18.04).

Remember to select these software selections under installation:
* docker
* OpenSSH server
* standard system utilities


## Upgrade Home Assistant
```bash
photon:~ keyne$ ssh -l carolee 192.168.112.10
carolee@goodgold:~$ screen
carolee@goodgold:~$ sudo -u homeassistant -H -s
[sudo] password for carolee: 

homeassistant@goodgold:/home/carolee$ source /srv/homeassistant/bin/activate
(homeassistant) homeassistant@goodgold:/home/carolee$ pip3 install --upgrade homeassistant
```

## Starting Home Assistant
```bash
photon:~ keyne$ ssh -l carolee 192.168.112.10
carolee@goodgold:~$ screen
carolee@goodgold:~$ sudo -u homeassistant -H -s
[sudo] password for carolee: 

homeassistant@goodgold:/home/carolee$ source /srv/homeassistant/bin/activate
(homeassistant) homeassistant@goodgold:/home/carolee$ hass
[Press Ctrl+a, then press d to detach from screen]
```

## Upgrading pip
```bash
photon:~ keyne$ ssh -l carolee 192.168.112.10
carolee@goodgold:~$ screen
carolee@goodgold:~$ sudo -u homeassistant -H -s
[sudo] password for carolee: 

homeassistant@goodgold:/home/carolee$ source /srv/homeassistant/bin/activate
(homeassistant) homeassistant@goodgold:/home/carolee$ pip install --upgrade pip
```

## Acknowledgments
* [https://www.home-assistant.io/components/logger/](https://www.home-assistant.io/components/logger/)
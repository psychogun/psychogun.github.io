---
layout: default
title: Network UPS Tools
parent: pfSense
nav_order: 2
---
# Network UPS Tools (NUT)
{: .no_toc }
This is how I used Network UPS Tools (NUT) for pfSense through USB and safely shut down my firewall and remotely told my <kbd>Proxmox</kbd> host to also shut down.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}
---
## Getting started

## Prerequisites
* pfSense 2.4.5-RELEASE-p3 (amd64)
* nut 2.7.4_7
* Proxmox 6.7
* PowerWalker VI 850 LCD


Using a powermeter I found out that the PowerWalker VI 850 LCD power consumption was around 13.7 W

pfSense 7,5W



## Install NUT
Navigate to <kbd>System</kbd> `>` <kbd>Packages</kbd> and select <kbd>Available Packages</kbd>, scroll down to the <kbd>NUT</kbd> package and click <kbd>Install</kbd>.

In order to make it correctly pick up the usb connection, you will have to reboot the firewall. Go to <kbd>Diagnostics</kbd> and hit <kbd>Reboot</kbd>.


## Services / UPS / Settings

### General Settings
* UPS Type: Local USB
* UPS Name: PowerWalker_VI_850_LCD
* Notifications: [v] Enable E-Mail notifications

### Driver Settings
* Driver: blazer

* Extra Arguments to driver (optional): 
```bash
desc="PowerWalker Line Interactive 850 LCD"
ignorelb
override.battery.charge.warning = 90
override.battery.charge.low = 85
```
The extra parameters `override.battery.charge.warning = 90` and `override.battery.charge.low = 85` will give a warning when the charge of the battery is below 90%. If the charge drops below 85%, it will send out shutdown commands. 


The <kbd>NUT</kbd> package will create two users for you, [admin] and [local-monitor]:
```bash
more /usr/local/etc/nut/upsd.users
[admin]
password=437219huadshfe04
actions=set
instcmds=all

[local-monitor]
password=238ruhfE2849dfasdfaad
upsmon master
```

## Fault finding
### upscmd
SSH in to your pfsense firewall, open up Shell (8):

Show the list of supported instant commands on our UPS named "PowerWalker_VI_850_LCD":
```bash
upscmd -l PowerWalker_VI_850_LCD
```
Use a user which has sufficient permissions, `instcmds=all`, to send a command to the UPS:
```bash
upscmd -u admin -p 437219huadshfe04 PowerWalker_VI_850_LCD <command>
```


## Authors
Mr. Johnson

## Acknowledgments
* [https://www.howtoforge.com/monitoring-ups-power-status-with-nut-on-opensuse10.3](https://www.howtoforge.com/monitoring-ups-power-status-with-nut-on-opensuse10.3)
* [https://networkupstools.org/docs/man/upscmd.html](https://networkupstools.org/docs/man/upscmd.html)
* [https://www.reddit.com/r/PFSENSE/comments/9pk3b9/usb_ups_with_nut_on_pfsense_master_and_synology/](https://www.reddit.com/r/PFSENSE/comments/9pk3b9/usb_ups_with_nut_on_pfsense_master_and_synology/)
* [https://nguvu.org/pfsense/network%20ups%20tools%20(nut)/pfsense-ups-nut/](https://nguvu.org/pfsense/network%20ups%20tools%20(nut)/pfsense-ups-nut/)
* [https://github.com/networkupstools/nut/issues/483#issuecomment-590977267](https://github.com/networkupstools/nut/issues/483#issuecomment-590977267)
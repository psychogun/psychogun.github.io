---
layout: default
title: Network UPS Tools
parent: pfSense
nav_order: 3
---
# Network UPS Tools (NUT)
{: .no_toc }
This is how I used Network UPS Tools (NUT) for pfSense through USB and safely shut down my firewall (acting as a netserver) and remotely told my <kbd>Proxmox</kbd> host to also shut down.

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
* pfSense 2.4.5-RELEASE-p3 (amd64)
* nut 2.7.4_7
* Proxmox 6.7
* PowerWalker VI 850 LCD


A quick and easy formula for calculating UPS runtime is, although not extremely precise: 

<kbd>t = V * Ah / W</kbd>

PowerWalker VI 850 has a 12V 9Ah battery.


Using a powermeter, I have calculated that the power consumption of the equipment is about 110 W.

Maximum runtime would thus be 0.98 hours with a 110 W drain:

`t = 12 * 9 / 110 = 0.98`


---

## Install NUT
Navigate to <kbd>System</kbd> `>` <kbd>Packages</kbd> and select <kbd>Available Packages</kbd>, scroll down to the <kbd>NUT</kbd> package and click <kbd>Install</kbd>.

In order to make it correctly pick up the usb connection, you will have to reboot the firewall. Go to <kbd>Diagnostics</kbd> and hit <kbd>Reboot</kbd>.

---

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


The <kbd>NUT</kbd> package will create two users for you, `admin` and `local-monitor`:
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

---

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

## Proxmox as netserver

```bash
nano /etc/nut/nut.conf
MODE=netserver
```

```bash
nano /etc/nut/ups.conf

[powerwalker]
    driver = blazer_usb
    port = auto
    runtimecal = 486,100,1296,50
# Remove maxretry as it is not supported by this driver
#maxretry=3
```
### Runtimecal
`runtimecal = value,value,value,value`

Parameter used in the (optional) runtime estimation. This takes two runtimes at different loads. Typically, this uses the runtime at full load and the runtime at half load. For instance, if your UPS has a rated runtime of 240 seconds at full load and 720 seconds at half load, you would enter

runtimecal = 240,100,720,50

PowerWalker 850 is rated for 480 watts.

Remember the formula <kbd>t = V * Ah / W</kbd>?

<kbd>t = 12 * 9 / 480 = 0.225 hours</kbd>, which is <kbd>13 minutes and 30 seconds (60 * 0.225)</kbd>, which is <kbd>13 * 60 = 810 seconds</kbd>

<kbd>t = 12 * 9 / 240 = 0.45 hours</kbd>, which is <kbd>27 minutes (60 * 0.45)</kbd>, which is <kbd>27 * 60 = 1620 seconds</kbd>

So I would have a `runtimecal` for this setup, and lets throw in a margin here of 0.6 and 0.8; 

<kbd>486,100,1296,50</kbd>

---

## Authors
Mr. Johnson

---

## Acknowledgments

* [https://www.howtoforge.com/monitoring-ups-power-status-with-nut-on-opensuse10.3](https://www.howtoforge.com/monitoring-ups-power-status-with-nut-on-opensuse10.3)
* [https://networkupstools.org/docs/man/blazer_usb.html](https://networkupstools.org/docs/man/blazer_usb.html)
* [https://electrocircuits.org/calculating-runtime-of-power-inverter-or-ups](https://electrocircuits.org/calculating-runtime-of-power-inverter-or-ups)
* [https://www.powerinspired.com/ups-runtime-calculator/](https://www.powerinspired.com/ups-runtime-calculator/)
* [http://www.company7.com/library/C7_upscalc.html](http://www.company7.com/library/C7_upscalc.html)
* [https://www.howtoforge.com/monitoring-ups-power-status-with-nut-on-opensuse10.3](https://www.howtoforge.com/monitoring-ups-power-status-with-nut-on-opensuse10.3)
* [https://networkupstools.org/docs/man/upscmd.html](https://networkupstools.org/docs/man/upscmd.html)
* [https://www.reddit.com/r/PFSENSE/comments/9pk3b9/usb_ups_with_nut_on_pfsense_master_and_synology/](https://www.reddit.com/r/PFSENSE/comments/9pk3b9/usb_ups_with_nut_on_pfsense_master_and_synology/)
* [https://nguvu.org/pfsense/network%20ups%20tools%20(nut)/pfsense-ups-nut/](https://nguvu.org/pfsense/network%20ups%20tools%20(nut)/pfsense-ups-nut/)
* [https://github.com/networkupstools/nut/issues/483#issuecomment-590977267](https://github.com/networkupstools/nut/issues/483#issuecomment-590977267)
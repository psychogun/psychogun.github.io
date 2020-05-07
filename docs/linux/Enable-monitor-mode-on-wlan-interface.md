---
layout: default
title: Enable monitor mode on wlan interface
parent: Linux
nav_order: 4
---
# How to enable monitor mode on wlan interface
{: .no_toc }
I wanted to create a better WiFi network for my self, thus setting my 802.11 wireless card in monitor mode I can discover the number of WiFi devices currently being used in my area.

This helped me reduce interference with other WiFi devices by choosing the least used WiFi channel for my Access Points. 

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}
---
## Getting started
You would need to have a computer running a flavour of linux and have a WiFi card (chipset) that supports monitor mode. 

### Prerequisites
* TP-Link Archer T9UH v2.0
* Linux parrot 5.2.0-2parrot1-amd64 SMP Debian 5.2.9-2parrot1 (2019-08-25) x86_64 GNU/Linux

## Does your card support monitoring mode?
Look after `monitor` under `Supported interface modes:`:
```bash
┌─[black@mamba]─[/etc]
└──╼ $iw list | more
(...)
Wiphy phy1
(...)
	Supported interface modes:
		 * IBSS
		 * managed
		 * AP
		 * monitor
```

But which wlan interface is `phy1`? Use `iw dev`:
```bash
┌─[black@mamba]─[/etc]
└──╼ $iw dev 
phy#1
        Interface wlan1
		       ifindex 5
			   wdev 0x3
			   addr
			   type managed
```

```bash
┌─[black@mamba]─[/etc]
└──╼ $sudo airmon-ng 

PHY	Interface	Driver		Chipset

phy0	wlan0		iwlwifi		Intel Corporation Centrino Ultimate-N 6300 (rev 3e)
phy1	wlan1		88XXau		TP-Link 802.11ac NIC
```
Our `phy1` interface is `wlan1`. Smooth sailing. 

## nmcli
```bash
┌─[✗]─[black@mamba]─[/etc]
└──╼ $nmcli device wifi
```

## airodump-ng
```bash
┌─[✗]─[black@mamba]─[/etc]
└──╼ $sudo airmon-ng start wlan1
```
```bash
┌─[black@mamba]─[/etc]
└──╼ $iw dev 
phy#1
        Interface wlan1
		       ifindex 5
			   wdev 0x3
			   addr
			   type monitor
```

```bash
┌─[✗]─[black@mamba]─[/etc]
└──╼ $sudo airodump-ng wlan1mon
```

### Come back
```bash
┌─[✗]─[black@mamba]─[/etc]
└──╼ $sudo airmon-ng stop wlan0mon
┌─[✗]─[black@mamba]─[/etc]
└──╼ $sudo service network-manager restart
┌─[✗]─[black@mamba]─[/etc]
└──╼ $sudo service wpa_supplicant restart
```



## Acknowledgments
* [https://forums.kali.org/showthread.php?28932-Avoiding-Airmon-ng-Check-Kill-and-restarting-NetworkManager](https://forums.kali.org/showthread.php?28932-Avoiding-Airmon-ng-Check-Kill-and-restarting-NetworkManager)
* [https://unix.stackexchange.com/questions/162088/why-airmon-ng-does-not-create-a-monitoring-interface](https://unix.stackexchange.com/questions/162088/why-airmon-ng-does-not-create-a-monitoring-interface)
* [https://superuser.com/questions/592296/using-iw-to-add-a-virtual-wireless-interface-getting-the-error-no-such-device](https://superuser.com/questions/592296/using-iw-to-add-a-virtual-wireless-interface-getting-the-error-no-such-device)
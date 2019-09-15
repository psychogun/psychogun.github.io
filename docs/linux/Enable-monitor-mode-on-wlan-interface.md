---
layout: default
title: Enable monitor mode on wlan interface
parent: Linux
nav_order: 5
---
# How to enable monitor mode on wlan interface
I wanted to create a better WiFi network for my self, thus setting my 802.11 wireless card in monitor mode I can discover the number of WiFi devices currently being used in my area. This helped me reduce interference with other WiFi devices by choosing the least used WiFi channel for my Access Points. 
## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}
---
# Getting started
You would need to have a computer able to run a WiFi card that supports monitor mode. 
# Prerequisites
* TP-Link Archer T9UH v2.0
* Linux parrot 5.2.0-2parrot1-amd64 #1 SMP Debian 5.2.9-2parrot1 (2019-08-25) x86_64 GNU/Linux
# Does your card support monitoring mode?
Look after `monitor` under `Supported interface modes:`:
```bash
┌─[black@mamba]─[/etc]
└──╼ $iw list | more
Wiphy phy1
	max # scan SSIDs: 9
	max scan IEs length: 2304 bytes
	max # sched scan SSIDs: 0
	max # match sets: 0
	max # scan plans: 1
	max scan plan interval: -1
	max scan plan iterations: 0
	Retry short limit: 7
	Retry long limit: 4
	Coverage class: 0 (up to 0m)
	Supported Ciphers:
		* WEP40 (00-0f-ac:1)
		* WEP104 (00-0f-ac:5)
		* TKIP (00-0f-ac:2)
		* CCMP-128 (00-0f-ac:4)
	Available Antennas: TX 0x4 RX 0x4
	Supported interface modes:
		 * IBSS
		 * managed
		 * AP
		 * monitor
```

But which wlan interface is `phy1`? Use `airmon-ng`:
```bash
┌─[black@mamba]─[/etc]
└──╼ $sudo airmon-ng 

PHY	Interface	Driver		Chipset

phy0	wlan0		iwlwifi		Intel Corporation Centrino Ultimate-N 6300 (rev 3e)
phy1	wlan1		88XXau		TP-Link 802.11ac NIC
```
Our `phy1` interface is `wlan1`. Smooth sailing. 

# Enable monitor mode
```bash
┌─[✗]─[black@mamba]─[/etc]
└──╼ $sudo ifconfig wlan1 down
┌─[black@mamba]─[/etc]
└──╼ $sudo iwconfig wlan1 mode monitor
┌─[black@mamba]─[/etc]
└──╼ $sudo ifconfig wlan1 up
┌─[black@mamba]─[/etc]
└──╼ $iwconfig
eth0      no wireless extensions.

wlan1     IEEE 802.11b  ESSID:""  Nickname:"<WIFI@REALTEK>"
          Mode:Monitor  Frequency:2.484 GHz  Access Point: Not-Associated   
          Sensitivity:0/0  
          Retry:off   RTS thr:off   Fragment thr:off
          Power Management:off
          Link Quality=0/100  Signal level=-100 dBm  Noise level=0 dBm
          Rx invalid nwid:0  Rx invalid crypt:0  Rx invalid frag:0
          Tx excessive retries:0  Invalid misc:0   Missed beacon:0

wlan0     IEEE 802.11  ESSID:"Pretty Fly For A 5.0GHz WiFi"  
          Mode:Managed  Frequency:5.1 GHz  Access Point: 14:CB:41:8C:9A:11   
          Bit Rate=195 Mb/s   Tx-Power=15 dBm   
          Retry short limit:7   RTS thr:off   Fragment thr:off
          Power Management:off
          Link Quality=69/70  Signal level=-41 dBm  
          Rx invalid nwid:0  Rx invalid crypt:0  Rx invalid frag:0
          Tx excessive retries:0  Invalid misc:235   Missed beacon:0

lo        no wireless extensions.

┌─[black@mamba]─[/etc]
└──╼ $
```
# Acknowledgments
* [https://unix.stackexchange.com/questions/162088/why-airmon-ng-does-not-create-a-monitoring-interface](https://unix.stackexchange.com/questions/162088/why-airmon-ng-does-not-create-a-monitoring-interface)
* [https://superuser.com/questions/592296/using-iw-to-add-a-virtual-wireless-interface-getting-the-error-no-such-device](https://superuser.com/questions/592296/using-iw-to-add-a-virtual-wireless-interface-getting-the-error-no-such-device)
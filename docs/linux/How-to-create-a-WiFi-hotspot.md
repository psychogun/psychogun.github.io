---
layout: default
title: How to create a WiFi hotspot
parent: Linux
nav_order: 5
---

# How to create a WiFi hotspot
{: .no_toc }
I wanted to use a Raspberry Pi 3+ to create a WiFi hotspot. 

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}
---

## Getting started
This is how I created a WiFi hotspot on my `wlan0` interface with a DHCP server and DNS, that would send internet traffic out on interface `eth0`.

### Prerequisites
* Kali Linux 4.19.66-Re4son-v7+
* Raspberry Pi 3+

## Installation
Install hostapd (hotspot server) and dnsmasq (dns and dhcp server):
```bash
root@kali:~# apt get install hostapd dnsmasq
```
Prevent the installed services starting at the boot:
```bash
root@kali:~# service hostapd stop
root@kali:~# service dnsmasq stop
root@kali:~# sudo update-rc.d hostapd disable
root@kali:~# sudo update-rc.d dnsmasq disable
```

## dnsmasq.conf
Secure a copy of the original `dnsmasq.conf` file:
```bash
root@kali:~# mv /etc/dnsmasq.conf /etc/dnsmasq.conf.bak
```
Create `dnsmasq.conf` from scratch (take a look at the original `dnsmasq.conf.bak` file to see all options you have):
```bash
root@kali:~# nano /etc/dnsmasq.conf

# Never forward plain names (without a dot or domain part)
domain-needed
# Never forward addresses in the non-routed address spaces.
bogus-priv
# Bind
bind-dynamic
# Forward DNS to google
server=8.8.8.8
# Supply the range of addresses available for lease and 
# optionally a lease time (12 hrs)
dhcp-range=192.168.150.10,192.168.150.30,255.255.255.0,12h
# If you want dnsmasq to listen for DHCP and DNS requests only on
# specified interfaces (and the loopback) give the name of the
# interface (eg eth0) here.
# Repeat the line for more than one interface.
interface=wlan0
```

## hostapd.conf
Create and setup the configuration file of `hostapd`:
```bash
root@kali:~# nano /etc/hostapd.conf

# Select the hotspot interface
interface=wlan0
# Name of WiFi
ssid=BerrySpot
# Set access point hardware mode to 802.11n (e.g. 2.4GHz)
hw_mode=g
# Meaning
ieee80211n=1
# Select WiFi channel
channel=6
```

## hotspot.sh
Create `hotspot.sh`:
```bash
root@kali:~# nano hotspot-wlan0.sh

#!/bin/bash
#Start
sudo ifconfig wlan0 192.168.150.1
sudo service dnsmasq restart
sudo sysctl net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo hostapd /etc/hostapd.conf
#Stop
sudo iptables -D POSTROUTING -t nat -o eth0 -j MASQUERADE
sudo sysctl net.ipv4.ip_forward=0
sudo service dnsmasq stop
sudo service hostapd stop
```
Make `hotspot.sh` executable:
```bash
root@kali:~# chmod +x hotspot.sh
```
Execute the script `hotspot.sh` and go on your other WiFi device to see if it is up:
```bash
root@kali:~# ./hotspot-wlan0.sh
```

## Acknowledgments
* [https://techstarspace.wordpress.com/2018/01/04/create-open-hotspot-on-kali-linux/](https://techstarspace.wordpress.com/2018/01/04/create-open-hotspot-on-kali-linux/)
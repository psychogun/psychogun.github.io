---
layout: default
title: Airodump with GPS coordinates
parent: Linux
nav_order: 1
---
# Airodump with GPS coordinates
{: .no_toc }
This is how I could use `airodump-ng` and walk around with my Raspberry Pi 3 and get an exact mapping in Google Earth of when/where `airodump-ng` discovered APs, amongst other devices.

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
Hook your Raspberry Pi 3 to an external battery source. To calculate how long your battery pack will power the device, go to [https://spellfoundry.com/raspberry-pi-battery-runtime-calculator/](https://spellfoundry.com/raspberry-pi-battery-runtime-calculator/). To calculate mAh on you battery, go to [https://www.omnicalculator.com/other/battery-capacity](https://www.omnicalculator.com/other/battery-capacity).

### Prerequisites
* Raspberry Pi Model 3
* Kali Linux
* Globalsat BU-353S4
* Battery pack

### Current USB devices
List your current USB devices.
```bash
root@kali:~# lsusb  
Bus 001 Device 005: ID 2357:0106 TP-Link Avision AM3000/MF3000 Series
Bus 001 Device 003: ID 0424:ec00  Avision AM3000/MF3000 Series
Bus 001 Device 002: ID 0424:9514  Avision AM3000/MF3000 Series
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
root@kali:~# 
```
Do you have a `ttyUSB0` device?
```bash
root@kali:~# ls /dev | grep USB
```

### Plug in your USB GPS dongle
See that is has been detected:
```bash
root@kali:~# lsusb 
Bus 001 Device 006: ID 067b:2303 Prolific Technology, Inc. PL2303 Serial Port
```
Has it shown up as `ttyUSB0`?
```bash
root@kali:~# ls /dev | grep USB
crw-rw----   1 root dialout 188,   0 Sep 21 20:33 ttyUSB0
```

---

## gpsd
Install an interface daemon for GPS receivers, `gpsd`:
```bash
root@kali:~# apt install gpsd
```
Run `gpsd`:
```bash
root@kali:~# screen
root@kali:~# gpsd -N -n -D 3 /dev/ttyUSB0
```
(Ctrl + a then d, to detach from screen).

NOTE: The `-N` option makes `gpsd` run in the foreground and the `-D` sets the debug level. 
This allows us to make sure the gps actually gets connected to the satellite.

---

## airodump-ng
Put your monitor mode compatible WiFi card in monitor mode:
```bash
root@kali:~# airmon-ng start wlan0
```
And then start `airodump-ng` with these parameters:
```bash
root@kali:~# screen
root@kali:~# airodump-ng wlan0mon -w packets --gpsd --output-format netxml -M -W -U
```
(Ctrl + a then d, to detach from screen).

-w : write to file called 'packets'
--gpsd : obtain gps data from gpsd
--output-format : use netxml for the -w file
-M : show a section with manufacturer
-W : show a coloumn with WPS technology
-U : show a coloumn with uptime obtained from AP beacon

PS: To se avilable parameters, check out `man airodump-ng`.

Then unplug your Raspberry Pi 3 from your ethernet connection and take a hike!

When you come back, plug your Raspberry Pi 3 in to your network, log on, resume your `airodump-ng` session with `screen -r` and use 'Ctrl + c' to stop the process. You'll now have a file in the directory from which `airodump-ng` was started, namely `packets-01.kismet.netxml`. 

---

## Google Earth
We can import the `packets-01.kismet.netxml` generated file into Google Earth, and have a layout of our airodumped APs with "exact" GPS coordinates with the help of a python script. But in order to make the python script work, we'll have to open the `packets-01.kismet.netxml` file in our favourite editor and replace symbols; 
```xml
                        <essid cloaked="true">&#x   0;&#x   0;&#x   0;&#x   0;&#x   0;&#x   0;&#x   0;&#x   0;&#x   0;&#x   0;&#x   0;&#x   0;&#x   0;&#x   0;&#x   0;&#x   0;&
#x   0;&#x   0;&#x   0;&#x   0;&#x   0;&#x   0;&#x   0;&#x   0;&#x   0;&#x   0;&#x   0;&#x   0;</essid>
```
To something like
```xml
                        <essid cloaked="true">0     0       0       0       0       0       0       0       0       0       0       0       0       0       0       0       0       0       0       0       0       0       0       0       0       0       0       0       0       0/essid>
```
.. then the following python script will be able to parse the .kismet.netxml format;

```bash
kali@root:~# wget https://files.salecker.org/netxml2kml/netxml2kml.py.txt
kali@root:~# mv netxml2kml.py.txt netxml2kml.py
(kali@root:~# chmod +x netxml2kml.py)
root@kali:~# python netxml2kml.py --kml packets-01.kismet.netxml -o WDRIVE
Parser: packets-01.kismet.netxml, 6504 new, 13 old
Outputfile: WDRIVE.*

KML export...
WPA     3782
WEP     25
None    304
Other   2393
Done. 6504 networks
root@kali:~# 
```
Then open up Chrome webbrowser, go to [https://earth.google.com](https://earth.google.com) and press 'Launch Google Earth in Chrome' > My Places > and click 'IMPORT KML FILE' and select `WDRIVE.kml`.

---

## Authors
Mr. Johnson

---

## Acknowledgments 
* [https://www.systutorials.com/docs/linux/man/8-gpsd/](https://www.systutorials.com/docs/linux/man/8-gpsd/)
* [https://www.question-defense.com/2011/02/18/creating-wireless-recon-maps-with-google-earth-kismet-gpsd-and-backtrack](https://www.question-defense.com/2011/02/18/creating-wireless-recon-maps-with-google-earth-kismet-gpsd-and-backtrack)
* [https://cqlug.linux.org.au/node/38](https://cqlug.linux.org.au/node/38)
* [http://www.teambsf.com/](http://www.teambsf.com/)
* [https://www.salecker.org/software/netxml2kml.html](https://www.salecker.org/software/netxml2kml.html)
* [https://charlesreid1.com/wiki/Airodump](https://charlesreid1.com/wiki/Airodump)

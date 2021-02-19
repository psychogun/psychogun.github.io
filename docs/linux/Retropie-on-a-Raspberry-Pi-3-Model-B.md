---
layout: default
title: Retropie on a Raspberry Pi 3 Model B
parent: Linux
nav_order: 17
---
# Retropie on a Raspberry Pi 3 Model B
{: .no_toc }
This is how I installed Retropie on Raspberry Pi. 

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

### Download premade image.

[https://retropie.org.uk/download/#Pre-made_images_for_the_Raspberry_Pi](https://retropie.org.uk/download/#Pre-made_images_for_the_Raspberry_Pi)

### Enable SSH

From a system with an SD-card reader, access the `/boot/` directory and create an empty file called `ssh`.
```bash
super:~ teddy$ cd /Volumes/boot
super:boot teddy$ touch ssh
```

### Enable WiFi
```bash
super:boot teddy$ nano wpa_supplicant.conf
country=FR # Your 2-digit country code
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
network={
    ssid="YOUR_NETWORK_NAME"
    psk="YOUR_PASSWORD"
    key_mgmt=WPA-PSK
}
```

---

## Authors
Mr. Johnson

---

## Acknowledgments
* [https://retropie.org.uk/forum/topic/19769/xsession-unable-to-start-x-session/4](https://retropie.org.uk/forum/topic/19769/xsession-unable-to-start-x-session/4)
* [https://cordcutting.com/how-to/how-to-install-kodi-on-retropie/](https://cordcutting.com/how-to/how-to-install-kodi-on-retropie/)
* [https://retropie.org.uk/docs/SSH/](https://retropie.org.uk/docs/SSH/)
* [https://howchoo.com/g/ndy1zte2yjn/how-to-set-up-wifi-on-your-raspberry-pi-without-ethernet](https://howchoo.com/g/ndy1zte2yjn/how-to-set-up-wifi-on-your-raspberry-pi-without-ethernet)
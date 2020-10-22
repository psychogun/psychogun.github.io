---
layout: default
title: Installation and configuration of Home Assistant
parent: Linux
nav_order: 8
---

# Installation and configuration of Home Assistant
{: .no_toc }
This is how I used Home Assistant to send notifications through Apple's Home app on my iOS devices, with the help of an <kbd>ubuntu-server</kbd> installation in a Proxmox Virtual Environment 6.1-5, with USB devices (Zigbee / Z-Wave dongles) passed through the host. 

I have some lights, a smartlock, some devices to measure temperatures with and, hm, what else; a motion detector that triggers the light if it is dark. This documentation is an attempt to gather all of my trials and notes when learning about Home Assistant. 


I am basically following this guide for the starting installation: [https://www.home-assistant.io/docs/installation/raspberry-pi/](https://www.home-assistant.io/docs/installation/raspberry-pi/)

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}
---

## Getting started
This is my guide. There are many like it, but this one is mine. My guide is my best friend. It is my life. 
I must master it as I must master my life. Without me, my guide is useless. Without my guide I am useless. I must trigger my guide true. 

### Prerequisites

**Software:**
* Proxmox Virtual Environment 6.1-5
* Ubuntu 20.04 LTS
* Home Assistant 0.113.3
* Deconz Docker

**Hardware:**
* Aeotec ZW090 Z-Stick Gen5 EU
* Aeotec ZW100 MultiSensor 6
* FIBARO System FGBS001 Universal Binary Sensor
* FIBARO System FGWPE/F Wall plug Gen5
* ID Lock AS ID Lock 150
* Philips Hue light
* IKEA TRADFRI light
* IKEA TRADFRI Remote
* Apple TV 4th gen

## Allocating a VM in Proxmox
Log in to your Proxmox in a web-browser and create a new virtal machine. 2GiB RAM and 32GiB harddrive is enough.

Power on the Virtual machine and follow this guide for an excellent guide to installing <kbd>ubuntu-server</kbd>: [https://tutorials.ubuntu.com/tutorial/tutorial-install-ubuntu-server-1604](https://tutorials.ubuntu.com/tutorial/tutorial-install-ubuntu-server-1604#0) 
(even though the link is for 1604, almost the same applies for 20.04).

Remember to select to install `OpenSSH server` under the installation of Ubuntu.

## Some things first ..
### Update && upgrade
Update and upgrade:
```bash
assistant@linuxbabe:~$ sudo apt-get update
assistant@linuxbabe:~$ sudo apt-get upgrade
```

### Set your time zone
```bash
assistant@linuxbabe:~$date
Sat 11 Jan 21:22:53 GMT 2020
assistant@linuxbabe:~$ sudo dpkg-reconfigure tzdata

Current default time zone: 'Europe/Paris'
Local time is now:      Sat Jan 11 22:24:07 CET 2020.
Universal Time is now:  Sat Jan 11 21:24:07 UTC 2020.

assistant@linuxbabe:~$ date
Sat 11 Jan 22:24:16 CET 2020
```

### Disable IPv6
```bash
assistant@linuxbabe:~$ sudo nano /etc/default/grub
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
assistant@linuxbabe:~$ sudo update-grub
```

### Change NTP 
Add your preferred NTP server:
```bash
assistant@linuxbabe:~$  sudo nano /etc/systemd/timesyncd.conf 
[sudo] password for assistant:
#  This file is part of systemd.
#
#  systemd is free software; you can redistribute it and/or modify it
#  under the terms of the GNU Lesser General Public License as published by
#  the Free Software Foundation; either version 2.1 of the License, or
#  (at your option) any later version.
#
# Entries in this file show the compile time defaults.
# You can change settings by editing this file.
# Defaults can be restored by simply deleting this file.
#
# See timesyncd.conf(5) for details.

[Time]
NTP=192.168.78.1
#FallbackNTP=ntp.ubuntu.com
#RootDistanceMaxSec=5
#PollIntervalMinSec=32
#PollIntervalMaxSec=2048 
```
Restart the `systemd-timesyncd` daemon:
```bash
assistant@linuxbabe:~$ sudo systemctl restart systemd-timesyncd
```

Check NTP status:
```bash
assistant@linuxbabe:~$ systemctl status systemd-timesyncd
● systemd-timesyncd.service - Network Time Synchronization
   Loaded: loaded (/lib/systemd/system/systemd-timesyncd.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2020-07-27 13:54:28 CEST; 50s ago
     Docs: man:systemd-timesyncd.service(8)
 Main PID: 626 (systemd-timesyn)
   Status: "Synchronized to time server 192.168.78.1:123 (192.168.78.1)."
    Tasks: 2 (limit: 2317)
   CGroup: /system.slice/systemd-timesyncd.service
           └─626 /lib/systemd/systemd-timesyncd

Jul 27 13:54:28 linuxbabe systemd[1]: Starting Network Time Synchronization...
Jul 27 13:54:28 linuxbabe systemd[1]: Started Network Time Synchronization.
Jul 27 13:54:32 linuxbabe systemd-timesyncd[626]: No network connectivity, watching for changes.
Jul 27 13:54:32 linuxbabe systemd-timesyncd[626]: No network connectivity, watching for changes.
Jul 27 13:54:32 linuxbabe systemd-timesyncd[626]: No network connectivity, watching for changes.
Jul 27 13:54:32 linuxbabe systemd-timesyncd[626]: No network connectivity, watching for changes.
Jul 27 13:54:32 linuxbabe systemd-timesyncd[626]: No network connectivity, watching for changes.
Jul 27 13:54:32 linuxbabe systemd-timesyncd[626]: No network connectivity, watching for changes.
Jul 27 13:54:59 linuxbabe systemd-timesyncd[626]: Synchronized to time server 192.168.78.1:123 (192.168.78.1).
assistant@linuxbabe:~$ 
```
### Qemu-guest-agent
This is only necessary if you are using <kbd>Proxmox</kbd> or another linux KVM: 
```bash
assistant@linuxbabe:~$ sudo apt-get install qemu-guest-agent
```
Issue `sudo shutdown now` to power of the guest and go to the <kbd>Proxmox</kbd> web gui and enable <kbd>QEMU Guest Agent</kbd> under <kbd>Options</kbd>, then start it again.


### python3 version
Which version do you currently have?
```bash
assistant@linuxbabe:~$ python3 -V
Python 3.8.2
```
#### (Optional) upgrade python
Install `python3.8` if you do not have the required version installed (`python3.7` and above is required- as of time writing):

```bash
assistant@linuxbabe:~$ python3 -V
Python 3.6.9
```

```bash
assistant@linuxbabe:~$ sudo apt-get install python3.8
```
The default `python3` environment is still `Python 3.6.9`.
```bash
assistant@linuxbabe:~$ python3 -V
Python 3.6.9
```

Add `python3.6` & `python3.8` to `update-alternatives`:
```bash
assistant@linuxbabe:~$ sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.6 1
update-alternatives: using /usr/bin/python3.6 to provide /usr/bin/python3 (python3) in auto mode
assistant@linuxbabe:~$ sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.8 2
update-alternatives: using /usr/bin/python3.8 to provide /usr/bin/python3 (python3) in auto mode
```
Update `python3` to point to `python3.8`:
```bash
assistant@linuxbabe:~$ sudo update-alternatives --config python3
There are 2 choices for the alternative python3 (providing /usr/bin/python3).

  Selection    Path                Priority   Status
------------------------------------------------------------
* 0            /usr/bin/python3.8   2         auto mode
  1            /usr/bin/python3.6   1         manual mode
  2            /usr/bin/python3.8   2         manual mode

Press <enter> to keep the current choice[*], or type selection number: 2
```
Test the version of `python`:
```bash
assistant@linuxbabe:~$ python3 -V
Python 3.8.0
```

### Install the dependencies
Install all the required dependencies for running Home Assistant in an `python` ***virtual environment***:
```bash
assistant@linuxbabe:~$ sudo apt-get install python3.8-dev python3.8-venv python3-pip libffi-dev libssl-dev libudev-dev build-essential
```
Add an account for Home Assistant called `homeassistant`. Since this account is only for running Home Assistant the extra arguments of `-rm` is added to create a system account and create a home directory. The arguments `-G dialout` adds the user to the `dialout` group. The `dialout` group is required for using Z-Wave and Zigbee controllers.

```bash
assistant@linuxbabe:~$ sudo useradd -rm homeassistant -G dialout
```
Next we will create a directory for the installation of Home Assistant and change the owner to the `homeassistant` account.

```bash
assistant@linuxbabe:~$ cd /srv
assistant@linuxbabe:/srv$ sudo mkdir homeassistant
assistant@linuxbabe:/srv$ sudo chown homeassistant:homeassistant homeassistant
```

Next up is to use the `homeassistant` user (`sudo -u homeassistant -H -s`) to create a virtual environment- in the current directory with `python3 -m venv .` and activate the `python` virtual environment (`source /srv/homeassistant/bin/activate`). This will be done as the `homeassistant` account:

```bash
assistant@linuxbabe:/srv$ sudo -u homeassistant -H -s
homeassistant@linuxbabe:/srv$ cd /srv/homeassistant
homeassistant@linuxbabe:/srv/homeassistant$ python3 -m venv .
homeassistant@linuxbabe:/srv/homeassistant$ source bin/activate
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ 
```

Once you have activated the virtual environment (notice the prompt change to `(homeassistant) homeassistant@linuxbabe:/srv/homeassistant $)` you will need to run the following command to install a required `python` package inside of the virtual environment (`wheel`):

```bash
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ python3 -m pip install wheel
Collecting wheel
  Downloading https://files.pythonhosted.org/packages/00/83/b4a77d044e78ad1a45610eb88f745be2fd2c6d658f9798a15e384b7d57c9/wheel-0.33.6-py2.py3-none-any.whl
Installing collected packages: wheel
Successfully installed wheel-0.33.6
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ 
```

Once you have installed the required `wheel` package it is now time to install Home Assistant in our virtual environment!

## Install Home Assistant
```bash
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ pip3 install homeassistant
```

Start Home Assistant for the first time. This will complete the installation for you, automatically creating the `.homeassistant` configuration directory in the `/home/homeassistant` directory, and installing any basic dependencies:

```bash
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ hass
```

When you run the `hass` command for the first time, it will download, install and cache the necessary libraries/dependencies. This procedure may take anywhere between 5 to 10 minutes. During that time, you may get “site cannot be reached” error when accessing the web interface. This will only happen for the first time, and subsequent restarts will be much faster.

After 5-10 minutes, you can now reach your installation over the web interface on `http://ipaddress:8123`. 

Hop on in and configure a user, set your timezone, location and metrics. It will also be possible to set up integrations discovered on the network. 

### Enable advanced mode
Click on your username on the down left. Enable Advanced Mode. 

### Home Assistant as a daemon
Stop `hass` with your keyboard, <kbd>CTRL</kbd> <kbd>+</kbd> <kbd>C</kbd> or through <kbd>Configuration</kbd>, <kbd>Server Controls</kbd> and <kbd>STOP</kbd>.

Do this if you would like Home Assistant to start on boot:
```bash
assistant@linuxbabe:/srv$ sudo nano /etc/systemd/system/home-assistant@homeassistant.service
[Unit]
Description=Home Assistant
After=network-online.target

[Service]
Type=simple
User=%i
ExecStart=/srv/homeassistant/bin/hass -c "/home/%i/.homeassistant"

[Install]
WantedBy=multi-user.target
```
You need to reload `systemd` to make the daemon aware of the new configuration:
```bash
assistant@linuxbabe:/srv$ sudo systemctl --system daemon-reload
```
To have Home Assistant start automatically at boot, enable the service.
```bash
assistant@linuxbabe:/srv$ sudo systemctl enable home-assistant@homeassistant
Created symlink /etc/systemd/system/multi-user.target.wants/home-assistant@homeassistant.service → /etc/systemd/system/home-assistant@homeassistant.service.
assistant@linuxbabe:/srv$ 
```
To disable the automatic start, use this command:
```bash
assistant@linuxbabe:/srv$ sudo systemctl disable home-assistant@homeassistant
Removed /etc/systemd/system/multi-user.target.wants/home-assistant@homeassistant.service.
```

To start Home Assistant **now**, use this command:
```bash
assistant@linuxbabe:/srv$ sudo systemctl start home-assistant@homeassistant
```

You can also substitute the `start` above with `stop` to stop Home Assistant, `restart` to restart Home Assistant, and `status` to see a brief status report as seen below:
```bash
assistant@linuxbabe:/srv$ sudo systemctl status home-assistant@homeassistant
```

To get Home Assistant’s logging output, simple use `journalctl`:
```bash
$ sudo journalctl -f -u home-assistant@homeassistant
```
Because the log can scroll quite quickly, you can select to view only the error lines with `grep`:
```bash
$ sudo journalctl -f -u home-assistant@homeasisstant | grep -i 'error'
```
When working on Home Assistant, you can easily restart the system and then watch the log output by combining the above commands using `&&`:
```
$ sudo systemctl restart home-assistant@YOUR_USER && sudo journalctl -f -u home-assistant@YOUR_USER
```

Do these things that are stated above/whatever you want, and shut down your VM guest through <kbd>Proxmox</kbd> web gui and/or issue `sudo shutdown now` inside the virtual machine.

## Attaching Z-Wave controllers
I am using a Aeotec ZW090 Z-Stick Gen5 EU for communication with my Z-Wave devices. We will have to pass USB device through the <kbd>Proxmox</kbd>host to the guest OS. 

First, list available USB devices on the proxmox host:
```bash
root@proxmox:~# lsusb 
Bus 002 Device 002: ID 8087:8000 Intel Corp. 
Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 001 Device 002: ID 8087:8008 Intel Corp. 
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 004 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
root@proxmox:~# 
```

Plug in the Z-Stick USB dongle:
```bash
root@proxmox:~# lsusb
Bus 002 Device 002: ID 8087:8000 Intel Corp. 
Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 001 Device 002: ID 8087:8008 Intel Corp. 
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 004 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 003 Device 002: ID 0658:0200 Sigma Designs, Inc. Aeotec Z-Stick Gen5 (ZW090) - UZB
Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
root@proxmox:~# 
```
All right, it is attached with and `ID` of `0658:0200`. 

It does not automatically attach itself in our Virtual Machine, as you would see if our installation was still powered on:
```bash
assistant@linuxbabe:~$ lsusb 
Bus 001 Device 002: ID 0627:0001 Adomax Technology Co., Ltd 
Bus 001 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
```
Select the Home Assistant Virtual Machine in Proxmox and go to <kbd>Hardware</kbd>. <kbd>Add</kbd>, <kbd>USB Device</kbd>. Select <kbd>Use USB Vendor/Device ID</kbd> and the device with the above ID (`Unknown (0658:0200)`).

Reboot/start the Virtual Machine. 

Now it is attached:
```bash
assistant@linuxbabe:~$ lsusb 
Bus 003 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 002 Device 002: ID 0658:0200 Sigma Designs, Inc. 
Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 001 Device 002: ID 0627:0001 Adomax Technology Co., Ltd 
Bus 001 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
```
To check where this USB device is mounted (<kbd>/dev/ttyACM0</kbd>) use:
```bash
assistant@linuxbabe:~$ ls -l /dev/serial/by-id/usb-
usb-0658_0200-if00                                                     usb-dresden_elektronik_ingenieurtechnik_GmbH_ConBee_II_DE2228251-if00  
assistant@linuxbabe:~$ ls -l /dev/serial/by-id/usb-dresden_elektronik_ingenieurtechnik_GmbH_ConBee_II_DE2228251-if00 
lrwxrwxrwx 1 root root 13 Oct 18 16:18 /dev/serial/by-id/usb-dresden_elektronik_ingenieurtechnik_GmbH_ConBee_II_DE2228251-if00 -> ../../ttyACM1
assistant@linuxbabe:~$ ls -l /dev/serial/by-id/usb-0658_0200-if00 
lrwxrwxrwx 1 root root 13 Oct 18 16:18 /dev/serial/by-id/usb-0658_0200-if00 -> ../../ttyACM0
```
You'll see that our device with ID `0658:0200` is `-> ../../ttyACM0`.

```bash
assistant@linuxbabe:~$ cd /sys/class/tty/
assistant@linuxbabe:/sys/class/tty$ readlink ttyACM0
../../devices/pci0000:00/0000:00:1e.0/0000:01:1b.0/usb2/2-1/2-1:1.0/tty/ttyACM0
```

After we have passed the USB device to our installation, we will further pass the USB device- to a docker running `ozwdaemon` which we will use to pair Z-Wave devices to our dongle with. 


## OpenZWave
<kbd>Promox</kbd>, <kbd>ubuntu-server</kbd>, <kbd>docker</kbd>, <kbd>ozwdaemon</kbd> - USB Z-Wave dongle. This is the setup which will allow messages from our Z-Wave enabled entities communicate with <kbd>Home Assistant</kbd> through a <kbd>mqtt</kbd> broker server installed directly on the <kbd>ubuntu-server</kbd>.

Let's do this.

### mosquitto
Before you can take usage of `ozwdaemon` docker to use it with our Home Assistant installation, we will have to install a `mqtt` broker. 

Follow this guide:
* [https://www.vultr.com/docs/how-to-install-mosquitto-mqtt-broker-server-on-ubuntu-16-04](https://www.vultr.com/docs/how-to-install-mosquitto-mqtt-broker-server-on-ubuntu-16-04)


### Portainer: prerequisites
We install the necessary packages to be able to install Docker:
```bash
assistant@linuxbabe:~$ sudo apt install apt-transport-https ca-certificates curl software-properties-common
```
We add the official Docker GPG key:
```bash
assistant@linuxbabe:~$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
OK
```
We activate Docker repository and update it:
```bash
assistant@linuxbabe:~$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
assistant@linuxbabe:~$ sudo apt update
```
We install the ***latest*** Docker version:
```bash
assistant@linuxbabe:~$ sudo apt install docker-ce
```

### Portainer: Installation
I use <kbd>Portainer</kbd> for managing Docker <kbd>containers</kbd>. Installing <kbd>Portainer</kbd> is very simple since it works in a Docker container, for this we will execute:
```bash
assistant@linuxbabe:~$ sudo docker volume create portainer_data
assistant@linuxbabe:~$ sudo docker run -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer
```

### Portainer: Admin user
Create an admin user through interface http://ip-adress:90000, after which you have opted for `Connect local`.

### Portainer: Make portainer start at boot
Go to Containers, select the portainer container and under `Container details`, select `Restart policies` and use `Unless stopped` and press `Update`. 

Connect ***locally***. 

* [https://community.home-assistant.io/t/home-assistant-add-on-openzwave/204034](https://community.home-assistant.io/t/home-assistant-add-on-openzwave/204034)

* [https://community.home-assistant.io/t/get-openzwave-beta-working/200121](https://community.home-assistant.io/t/get-openzwave-beta-working/200121)

#### Portainer: Upgrade Portainer
After a while, I wanted to update my <kbd>Portainer</kbd> docker.

First, list our containers:
```
assistant@linuxbabe:/$ sudo docker container ls
[sudo] password for assistant: 
CONTAINER ID        IMAGE                                 COMMAND             CREATED             STATUS              PORTS                                                                    NAMES
f6f62aef31b1        openzwave/ozwdaemon:allinone-latest   "/init"             10 minutes ago      Up 10 minutes       0.0.0.0:1983->1983/tcp, 0.0.0.0:5901->5901/tcp, 0.0.0.0:7800->7800/tcp   ozwdaemon
855186d94b34        portainer/portainer                   "/portainer"        2 months ago        Up 4 weeks          0.0.0.0:9000->9000/tcp                                                   mystifying_rubin
```
Stop our <kbd>Portainer</kbd> container:
```bash
assistant@linuxbabe:/$ sudo docker stop 855186d94b34
```
Remove portainer:
```bash
sudo docker rm 855186d94b34
```

Pull:
```bash
assistant@linuxbabe:/$ sudo docker pull portainer/portainer-ce
Using default tag: latest
latest: Pulling from portainer/portainer-ce
d1e017099d17: Already exists 
b0718b1ef1b0: Pull complete 
Digest: sha256:0ab9d25e9ac7b663a51afc6853875b2055d8812fcaf677d0013eba32d0bf0e0d
Status: Downloaded newer image for portainer/portainer-ce:latest
docker.io/portainer/portainer-ce:latest
```
Run:
```bash
assistant@linuxbabe:/$ sudo docker run -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce
423debb02a73ef7275e4e221e3e2fb276387b0ca00e12c271254ce4c4e03d424
```

### ozwdaemon: Prerequisites
```bash
assistant@linuxbabe:/$ cd /opt/
assistant@linuxbabe:/opt$ sudo mkdir ozw
assistant@linuxbabe:/opt$ cd ozw/
assistant@linuxbabe:/opt/ozw$ sudo mkdir config
```

#### ozwdaemon: Configuration
I configure my <kbd>ozwdaemon</kbd> Docker container through <kbd>Portainer</kbd>:

**Add Container:**
* Name: ozwdaemon
* Image: docker.io openzwave/ozwdaemon:allinone-latest

**Network ports configuration:**
* 1983:1983
* 5901:5901
* 7800:7800

**Volumes:**
+map additional volume
* container: /opt/ozw/config <click Bind>
* host: /opt/ozw/config

**Change time of container:**
* container: /etc/localtime <click Bind>
* host: /etc/localtime

**Env:**
* `MQTT_SERVER: "192.168.0.1"`
* `MQTT_USERNAME: "my-username"`
* `MQTT_PASSWORD: "my-password"`
* `USB_PATH: "/dev/ttyACM0"`
* `OZW_NETWORK_KEY: "0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00"`

<kbd>PS:</kbd> To generate a random Z-Wave Network Key:
```bash
cat /dev/urandom | tr -dc '0-9A-F' | fold -w 32 | head -n 1 | sed -e 's/\(..\)/0x\1, /g' -e 's/, $//'
```
<kbd>PSPS:</kbd>This key will be used throughout to pair your Z-Wave devices to your Z-Wave stick. Make sure to have a backup of this.

**Runtime & Resources:**
* host: /dev/ttyACM0
* container: /dev/ttyACM0

**Deplo**y the container.

Check [http://localhost:7800](http://localhost:7800) for VNC web-based configuration of `ozwdaemon`.


### Integrations
MQTT 

* Broker: localhost 

## Z-Wave

sensor.door_window_sensor_6_basic
## configuration.yaml
Here are som code snippets that I have used directly in the `configuration.yaml` file. 
### ID-Lock 150

#### ID Lock 150 configuration
Go to Configuration > Z-Wave > Select Node `ID Lock AS 150 (Node 7: Complete)
* Node Config Options: 3: Door Hinge Position
* Config Value: Left Handle
Click `SET CONFIG PARAMETER`.

* Node Config Options: 4: Door Audio Volume Level
* Config Value: No Sound
Click `SET CONFIG PARAMETER`.

Which user opened the Door?

```bash
# Which user opened the door
binary_sensor:
  - platform: template
    sensors:
# John
      id_150_user_code_john:
       friendly_name: "Johnson opened the door"
       value_template: >-
         {{ is_state('sensor.id_150_z_wave_module_user_code', '63') }}
# Sarah      
      id_150_user_code_sarah:
       friendly_name: "Sarah opened the door"
       value_template: >-
         {{ is_state('sensor.id_150_z_wave_module_user_code', '62') }}
# Michael
      id_150_user_code_michael:
       friendly_name:  "Michael opened the door"
       value_template: >-
         {{ is_state('sensor.id_150_z_wave_module_user_code', '61') }}
```

### Ping
Ping these devices. 
```bash
## Presence detector
device_tracker:
  - platform: ping
    interval_seconds: 30
    consider_home: 1200
    hosts:
# Mr. Robot
      robot_iphone: 172.16.32.3.56
```
Once it has established a ping, it will create a file called `known_devices.yaml` and in that file you can replace icon with a picture:
```bash
sudo nano known_devices.yaml

robot_iphone:
  hide_if_away: false
  icon:
  mac:
  name: Mr. Robot iPhone 6S
  picture:
  track: true
```

### Systemmonitor
```bash
# Example configuration.yaml entry
sensor:
  - platform: systemmonitor
    resources:
      - type: disk_use_percent
        arg: /home
      - type: memory_free
```

### kWh total cost
I have devices which have an entity which gives me total kWh used. Use this to do a calculation of the total running cost:
```bash
sensor:
  - platform: template
    sensors: 
# Bathroom Heat Cost Total
      bathroom_heat_cost_total:
        friendly_name: Bathroom Heat Cost Total
        icon_template: mdi:cash
        unit_of_measurement: 'EUR'
        value_template: '{{ ((states.sensor.general_thermostat_v2_electric_kwh_2.state)) | multiply(0.728) | round(2) }}'
# Entrance Heat Cost  Total
      entrance_heat_cost_total:
        friendly_name: Entrance Heat Cost Total
        icon_template: mdi:cash
        unit_of_measurement: 'EUR'
        value_template: '{{ ((states.sensor.general_thermostat_v2_electric_kwh.state)) | multiply(0.728) | round(2) }}'
```

### System Health
Enable System Health by adding `system_health:` in `configuration.yaml`, add it to the very top:

```bash
assistant@linuxbabe:/home/homeassistant/.homeassistant$ sudo nano configuration.yaml 
# Enable System Health
system_health:
```

## Integrations: Z-Wave
Different entities from my Z-Wave network. 

### Adding a tampering sensor
In the Home Assistant GUI, go to Configuration and select Entities. There you'll see that you have a sensor which is called `AEON Labs ZW100 MultiSensor 6 Burglar`. This is the motion sensor on the AEON Lab ZW100 device. Click on it, and you'll see what the entity is named: `sensor.aeon_labs_zw100_multisensor_6_burglar`. This gives us a value `3` for tampering with the device, `8` for motion and `254` for sleep. We're going to add a binary sensor to be able to get notifications in Apple Home for tampering.

```bash
assistant@linuxbabe:/home/homeassistant/.homeassistant$ sudo nano configuration.yaml 

binary_sensor:
  - platform: template
    sensors:
      burglar_up:
       friendly_name: "Tampering detector"
       value_template: >-
         "{{ is_state('sensor.aeotec_zw100_multisensor_6_burglar', '3') }}"
```
Wake up the Multisensor 6, as stated in the documentation found here [https://aeotec.freshdesk.com/support/solutions/articles/6000057073-multisensor-6-user-guide-](https://aeotec.freshdesk.com/support/solutions/articles/6000057073-multisensor-6-user-guide-):
* Press and hold Multisensor 6 Action button
* Wait until the RGB LED turns into a Yellow/Orange Color
* Release Multisensor 6 Action Button
* The LED on Multisensor 6 will now rapidly blink its Yellow/Orange LED while it is in its awake state. You may send in any configurations or commands from your current gateway to configure your Multisensor 6.

Then go to <kbd>Configuration</kbd>, <kbd>Z-Wave Node Management</kbd>. Select

* Nodes: AEON Labs ZW100 Multisensor 6
Entities of this node:
* binary_sensor.aeon_labs_zw100_multisensor_6_sensor
* Node Values: Burglag (Instance; 1, Index; 10)
* Node group associations: 
* Node Config Options: **3600**
* Config Parameter: 5; Command Options
* Config Value: ** Binary Sensor Report **

Click <kbd>SET CONFIG PARAMETER</kbd>.

* Tap the Action Button on Multisensor 6 to put Multisensor 6 back to sleep, or wait 10 minutes. (recommended to manually put it back to sleep to conserve battery life).

<kbd>PS:</kbd> Make sure that also the entity `binary_sensor.aeon_labs_zw100_multisensor_6_sensor` also uses `Binary Sensor Report`, as stated as best practise here: [https://www.home-assistant.io/docs/z-wave/entities/](https://www.home-assistant.io/docs/z-wave/entities/).

### Fantem (Oomi) Unknown: type=0002, id=0070
I have a door / window sensor from Fantem, which is not yet listed with configuration files on the OpenZwave project page. 

Anywhow, this is the same device as the Aeotec Zw112. So you could just link this configuration file to that device;

```bash
sudo nano /srv/homeassistant/lib/python3.8/site-packages/python_openzwave/ozw_config/manufacturer_specific.xml
(...)
<Manufacturer id="016a" name="Fantem (Oomi)">
(...)
                <Product type="0002" id="0070" name="ZW112 Door Window Sensor 6" config ="aeotec/zw112.xml" />
</Manufacturer>
```

To make it take effect, remove the `zwcfg_*.xml` inside of `/home/homeassistant/.homeassistant`.
```bash
sudo systemctl stop home-assistant@homeassistant
sudo mv zwcfg_0xac103ded.xml zwcfg_0xac103ded.backup
sudo systemctl start home-assistant@homeassistant
```

Go to <kbd>Configuration</kbd>, <kbd>Devices</kbd> and delete all the unknown entities. Restart `sudo systemctl restart home-assistant@homeassistant`. 

Go to <kbd>Configuration</kbd>, <kbd>Z-Wave</kbd> and <kbd>ADD NODE SECURE</kbd> and follow the inclusion settings for the device from the manual. Now it should show up as the ZW112 device. 

Anyhow, I ended up following this article upgrading the firmware of the Fantem (Oomi) sensor, as it's the same device: [https://aeotec.freshdesk.com/support/solutions/articles/6000228743-how-to-update-door-window-sensor-6-z-wave-firmware-](https://aeotec.freshdesk.com/support/solutions/articles/6000228743-how-to-update-door-window-sensor-6-z-wave-firmware-). It now shows up as AEON Labs ZW112 Door Window Sensor 6.

Go to <kbd>Configuration</kbd>, <kbd>Z-Wave</kbd>, selec <kbd>AEON Labs ZW112 Door Window Sensor 6 (Node:2 Complete)</kbd> and scroll down to <kbd>Node Configuration Options</kbd>. Set <kbd>121: Report Type</kbd> to <kbd>Send configuration parameter</kbd> to <kbd>Sensor Binary Report</kbd>. 

## Enable Multi-factor Authentication Modules
Go to your profile by clicking your name down to the left.

Under `Multi-factor Authentication Modules`, click ENABLE on `totp`. Download Google Authenticator from your App Store and scan the code. For more information, go to [https://www.home-assistant.io/docs/authentication/multi-factor-auth/](https://www.home-assistant.io/docs/authentication/multi-factor-auth/).

## Home Assistant Companion
Install Home Assistant Companion from your App Store, but only do it after you have enabled Multi-factor. Do not log in and use it before enabling multi-factor authentication, because you will be unable to log on in without destroying your entities (because you are unable to connect, because you have not used Google Authenticator under that process).

## Change icons
Change your icons by using `mdi:icon` - look at this list for reference: [https://cdn.materialdesignicons.com/3.2.89/](https://cdn.materialdesignicons.com/3.2.89/)


## Backup Home Assistant
More should come..

## Upgrade Home Assistant
```bash
assistant@linuxbabe:/home/homeassistant/.homeassistant$ cd /srv/
assistant@linuxbabe:/srv$ cd homeassistant
assistant@linuxbabe:/srv/homeassistant$ sudo -u homeassistant -H -s
[sudo] password for assistant: 
homeassistant@linuxbabe:/srv/homeassistant$ source bin/activate
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ 
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ systemctl | grep home
  home-assistant@homeassistant.service                                                               loaded active running   Home Assistant                                                               
  system-home\x2dassistant.slice                                                                     loaded active active    system-home\x2dassistant.slice             
```
Remember to stop `homeassistant` if it is running:
```bash
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ systemctl status home-assistant@homeassistant
● home-assistant@homeassistant.service - Home Assistant
   Loaded: loaded (/etc/systemd/system/home-assistant@homeassistant.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2020-03-02 00:08:17 CET; 1 months 3 days ago
 Main PID: 994 (hass)
    Tasks: 40 (limit: 3194)
   CGroup: /system.slice/system-home\x2dassistant.slice/home-assistant@homeassistant.service
           └─994 /srv/homeassistant/bin/python3 /srv/homeassistant/bin/hass -c /home/homeassistant/.homeassistant
```
Regardless of if Home Assistant was running or not, this is how I stopped it:
```bash
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ systemctl stop home-assistant@homeassistant
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-units ===
Authentication is required to stop 'home-assistant@homeassistant.service'.
Authenticating as: Home Assistant (assistant)
Password: 
==== AUTHENTICATION COMPLETE ===
```
Upgrade `homeassistant`:
```bash
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ pip3 install --upgrade homeassistant
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ systemctl start home-assistant@homeassistant
```

## Manual start of Home Assistant
```bash
photon:~ keyne$ ssh -l assistant 192.168.112.10
assistant@linuxbabe:~$ screen
assistant@linuxbabe:~$ sudo -u homeassistant -H -s
[sudo] password for assistant: 

homeassistant@linuxbabe:/home/assistant$ source /srv/homeassistant/bin/activate
(homeassistant) homeassistant@linuxbabe:/home/carolee$ hass
[Press Ctrl+a, then press d to detach from screen]
```

## Upgrading pip
```bash
photon:~ keyne$ ssh -l assistant 192.168.112.10
assistant@linuxbabe:~$ screen
assistant@linuxbabe:~$ sudo -u homeassistant -H -s
[sudo] password for assistant: 

homeassistant@linuxcbabe:/home/assistant$ source /srv/homeassistant/bin/activate
(homeassistant) homeassistant@linuxbabe:/home/carolee$ pip install --upgrade pip
```

## Camera
I have a Unifi NVR on my network. This i s how  I integrated it.

Stop Home Assistant:

```bash
assistant@linuxbabe:/home/homeassistant/.homeassistant$ cd /srv/
assistant@linuxbabe:/srv$ cd homeassistant
assistant@linuxbabe:/srv/homeassistant$ sudo -u homeassistant -H -s
[sudo] password for assistant: 
homeassistant@linuxbabe:/srv/homeassistant$ source bin/activate
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ 
```
Is  it running?
```bash
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ systemctl | grep home
  home-assistant@homeassistant.service                                                               loaded active running   Home Assistant                                                               
  system-home\x2dassistant.slice                                                                     loaded active active    system-home\x2dassistant.slice  
```
Stop it:
```bash                     
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ systemctl status home-assistant@homeassistant
● home-assistant@homeassistant.service - Home Assistant
   Loaded: loaded (/etc/systemd/system/home-assistant@homeassistant.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2020-03-02 00:08:17 CET; 1 months 3 days ago
 Main PID: 994 (hass)
    Tasks: 40 (limit: 3194)
   CGroup: /system.slice/system-home\x2dassistant.slice/home-assistant@homeassistant.service
           └─994 /srv/homeassistant/bin/python3 /srv/homeassistant/bin/hass -c /home/homeassistant/.homeassistant
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ systemctl stop home-assistant@homeassistant
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-units ===
Authentication is required to stop 'home-assistant@homeassistant.service'.
Authenticating as: Home Assistant (assistant)
Password: 
==== AUTHENTICATION COMPLETE ===
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$
```
* Follow this guide [https://www.home-assistant.io/integrations/uvc/](https://www.home-assistant.io/integrations/uvc/)

Install `ffmpeg`:
```bash
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ exit
assistant@linuxbabe:/srv/homeassistant$  sudo apt-get install ffmpeg
```

This is what I ended up with in my `configuration.yaml` file:
```bash

# UniFi Network Video Recorder
# https://www.home-assistant.io/integrations/uvc/
camera:
  - platform: uvc
    nvr: ip-address-to-NVR
    key: !secret uvc_api
    password: !secret password_to_camera
```

Start Home Assistant:
```bash
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ sudo -u homeassistant -H -s
assistant@linuxbabe:/srv/homeassistant$ systemctl start home-assistant@homeassistant
```

OK?
```bash
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ exit
assistant@linuxbabe:/srv/homeassistant$ sudo journalctl -f -u home-assistant@homeassistant
```

## Person tracker
Configuration > Customization

 entity_picture: URL

## Zigbee

### Conbee-II 
Upgrade Conbee II:
* [https://github.com/dresden-elektronik/deconz-rest-plugin/wiki/Update-deCONZ-manually#update-in-ubuntu](https://github.com/dresden-elektronik/deconz-rest-plugin/wiki/Update-deCONZ-manually#update-in-ubuntu)

Download firmware:
* [http://deconz.dresden-elektronik.de/deconz-firmware/?C=M;O=D](http://deconz.dresden-elektronik.de/deconz-firmware/?C=M;O=D)


deCONZ by dresden elektronik is a software that communicates with ConBee/RaspBee Zigbee gateways and exposes Zigbee devices that are connected to the gateway.

### ZHA 
There are different ways to connect your Zigbee devices to Home Assistant (Zigbee2mqtt, deCONZ, ZHA). I opted for using Zigbee Home Automation (the integrated one). 

Shut the proxmox guest from proxmox. Attach the USB physically, add it to the guest and start home assistant again. My device showed up as `FT230X Basic UART (0403:6015)` in proxmox. 

```bash
assistant@linuxbabe:~$ lsusb 
Bus 002 Device 003: ID 0403:6015 Future Technology Devices International, Ltd Bridge(I2C/SPI/UART/FIFO)
assistant@linuxbabe:~$ ls -alh /dev/ttyUSB0 
crw-rw---- 1 root dialout 188, 0 Feb  4 22:39 /dev/ttyUSB0
```
Following this guide [https://www.home-assistant.io/integrations/zha/](https://www.home-assistant.io/integrations/zha/)

```bash
ZHA
USB Device Path: /dev/ttyUSB0
Radio Type: deconz
```

### IKEA Tradfri
#### IKEA of Sweden TRADFRI on/off switch
Unscrew the lock. Go to Configuration > Integrations >  and click `CONFIGURE` on your ZHA integration and then the yellow `+` sign down on the right. Click 4 times (within 5 seconds (?)) on the reset button on the on/off switch. After 15 seconds or more it will show up as a new entity.

If you have more switches, I recommend marking them and changing the name of the switch / adding e.g. "_1" or sonething and mark the switch with a pen, so you know which ones is which.

#### IKEA TRADFRI Remote
This is the round one, with 5 buttons.

Open up the lid. Go to <kbd>Configuration</kbd>, <kbd>Integrations</kbd> and click ´CONDFIGURE` on your ZHA integration and then the yello `+`sign down on the right. Click 4 times within 5 seconds on the reset button on the on/off switch. After 15 seconds or more it will show up as a new entity. 

#### IKEA bulb
* [https://phoscon.de/en/support#faq-ikea1](https://phoscon.de/en/support#faq-ikea1)


## Samba share
Install Samba:
```bash
assistant@linuxbabe:~$ sudo apt-get install samba
```
Configure a user:
```bash
assistant@linuxbabe:~$ sudo smbpasswd -a homeassistant
New SMB password:
Retype new SMB password:
Added user homeassistant.
assistant@linuxbabe:~$ 
```
I chose the same username as the user in linux, and the same password as I have in linux.

Make a copy of the original `smb.conf` file:
```bash
assistant@linuxbabe:~$ sudo cp /etc/samba/smb.conf ~
```
<kbd>PS:</kbd> `~` is the current user's home directory.

Configure `smb.conf`:
```bash
assistant@linuxbabe:/home/homeassistant/.homeassistant$ sudo nano /etc/samba/smb.conf
# Allow users who've been granted usershare privileges to create
# public shares, not just authenticated ones
   usershare allow guests = no

# Un-comment the following (and tweak the other settings below to suit)
# to enable the default home directory shares. This will share each
# user's home directory as \\server\username
[homes]
  comment = Home Directories
;   browseable = no

# By default, the home directories are exported read-only. Change the
# next parameter to 'no' if you want to be able to write to them.
   read only = no

# File creation mask is set to 0700 for security reasons. If you want to
# create files with group=rw permissions, set next parameter to 0775.
   create mask = 0600

```

Restart the `samba` service:
```bash
assistant@linuxbabe:~$ sudo service smbd restart
```
Once Samba has restarted, use this command to check your smb.conf for any syntax errors:
```bash
assistant@linuxbabe:~$ testparm
Load smb config files from /etc/samba/smb.conf
rlimit_max: increasing rlimit_max (1024) to minimum Windows limit (16384)
WARNING: The "syslog" option is deprecated
Processing section "[homes]"
Loaded services file OK.
WARNING: The 'netbios name' is too long (max. 15 chars).

Server role: ROLE_STANDALONE

Press enter to see a dump of your service definitions
^C
```

On MacOS, open Finder and press `command + K`. Connect to server `smb://172.16.32.17/homeassistant/.homeassistant` to see your configuration files. 

Now you can directly edit the `configuration.yaml` file from MacOS!

## Fault finding
### ModuleNotFoundError: No module named 'apt_pkg'
```bash
assistant@linuxbabe:~$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
Traceback (most recent call last):
  File "/usr/bin/add-apt-repository", line 11, in <module>
    from softwareproperties.SoftwareProperties import SoftwareProperties, shortcut_handler
  File "/usr/lib/python3/dist-packages/softwareproperties/SoftwareProperties.py", line 28, in <module>
    import apt_pkg
ModuleNotFoundError: No module named 'apt_pkg'
```
```bash
assistant@linuxbabe:~$ ls -l /usr/lib/python3/dist-packages/apt_pkg*
-rw-r--r-- 1 root root 346784 Jan 24  2020 /usr/lib/python3/dist-packages/apt_pkg.cpython-36m-x86_64-linux-gnu.so
-rw-r--r-- 1 root root   8900 Jan 24  2020 /usr/lib/python3/dist-packages/apt_pkg.pyi
```


```bash
assistant@linuxbabe:~$ sudo update-alternatives --config python3
There are 2 choices for the alternative python3 (providing /usr/bin/python3).

  Selection    Path                Priority   Status
------------------------------------------------------------
  0            /usr/bin/python3.7   2         auto mode
  1            /usr/bin/python3.6   1         manual mode
* 2            /usr/bin/python3.7   2         manual mode

Press <enter> to keep the current choice[*], or type selection number:  <enter> 
update-alternatives: warning: forcing reinstallation of alternative /usr/bin/python3.7 because link group python3 is broken
```
```bash
assistant@linuxbabe:~$ sudo apt remove python3.6*
```

```bash
sudo apt install python3-apt
sudo apt install software-properties-common
```

```bash
assistant@linuxbabe:~$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
Hit:1 http://no.archive.ubuntu.com/ubuntu bionic InRelease
Hit:2 http://no.archive.ubuntu.com/ubuntu bionic-updates InRelease
Hit:3 http://no.archive.ubuntu.com/ubuntu bionic-backports InRelease                  
Hit:4 http://no.archive.ubuntu.com/ubuntu bionic-security InRelease                   
Get:5 https://download.docker.com/linux/ubuntu bionic InRelease [64.4 kB]             
Get:6 https://download.docker.com/linux/ubuntu bionic/stable amd64 Packages [12.5 kB]
Fetched 76.9 kB in 1s (147 kB/s)      
Reading package lists... Done
```
Not quite sure if I remember what I did to fix this. I think I just started from scratch. 


### HAP-python not found
```bash
assistant@linuxbabe:/home/homeassistant/.homeassistant$ tail -f home-assistant.log 
2020-01-26 14:22:43 ERROR (SyncWorker_2) [homeassistant.util.package] Unable to install package homeassistant-pyozw==0.1.7: Command "/srv/homeassistant/bin/python3 -u -c "import setuptools, tokenize;__file__='/tmp/pip-build-v_f92wss/homeassistant-pyozw/setup.py';f=getattr(tokenize, 'open', open)(__file__);code=f.read().replace('\r\n', '\n');f.close();exec(compile(code, __file__, 'exec'))" install --record /tmp/pip-pq6berwr-record/install-record.txt --single-version-externally-managed --compile --install-headers /srv/homeassistant/include/site/python3.7/homeassistant-pyozw" failed with error code 1 in /tmp/pip-build-v_f92wss/homeassistant-pyozw/
2020-01-26 14:22:47 ERROR (SyncWorker_1) [homeassistant.util.package] Unable to install package HAP-python==2.6.0: Command "/srv/homeassistant/bin/python3 -u -c "import setuptools, tokenize;__file__='/tmp/pip-build-3ljvhzbx/ed25519/setup.py';f=getattr(tokenize, 'open', open)(__file__);code=f.read().replace('\r\n', '\n');f.close();exec(compile(code, __file__, 'exec'))" install --record /tmp/pip-s_mwhh88-record/install-record.txt --single-version-externally-managed --compile --install-headers /srv/homeassistant/include/site/python3.7/ed25519" failed with error code 1 in /tmp/pip-build-3ljvhzbx/ed25519/
2020-01-26 14:22:47 ERROR (MainThread) [homeassistant.components.homeassistant] Component error: zwave - Requirements for zwave not found: ['homeassistant-pyozw==0.1.7'].
Component error: default_config - Requirements for homekit not found: ['HAP-python==2.6.0'].
2020-01-26 14:27:14 ERROR (SyncWorker_2) [homeassistant.util.package] Unable to install package homeassistant-pyozw==0.1.7: Command "/srv/homeassistant/bin/python3 -u -c "import setuptools, tokenize;__file__='/tmp/pip-build-6yvub58z/homeassistant-pyozw/setup.py';f=getattr(tokenize, 'open', open)(__file__);code=f.read().replace('\r\n', '\n');f.close();exec(compile(code, __file__, 'exec'))" install --record /tmp/pip-ll4z1ocf-record/install-record.txt --single-version-externally-managed --compile --install-headers /srv/homeassistant/include/site/python3.7/homeassistant-pyozw" failed with error code 1 in /tmp/pip-build-6yvub58z/homeassistant-pyozw/
2020-01-26 14:27:19 ERROR (MainThread) [homeassistant.components.homeassistant] Component error: zwave - Requirements for zwave not found: ['homeassistant-pyozw==0.1.7'].
```

```bash
assistant@linuxbabe:/home/homeassistant/.homeassistant$ pip3 install HAP-python
assistant@linuxbabe:/home/homeassistant/.homeassistant$ sudo apt-get install libudev-dev
```

### ID Lock 150
Nothing happens when I press UNLOCK/LOCK on my ID Lock 150. I was sure to be on the latest firmware. My ID Lock 150 is running the latest firmware version 1.5.0 ([https://idlock.no/kundesenter/idl150updater-version/](https://idlock.no/kundesenter/idl150updater-version/)).

I checked the logs and I saw this;
```bash
assistant@linuxbabe:/home/homeassistant/.homeassistant$ tail -f OZW_Log.txt 
2020-01-28 22:56:35.465 Error, Node008, ERROR: Dropping command, expected response not received after 1 attempt(s)
```

So I am going to try to do a Z-Wave local reset, as instructed in the manual for ID Lock 150, [https://idlock.no/wp-content/uploads/2019/08/IDLock150_ZWave_UserManual_v3.02.pdf](https://idlock.no/wp-content/uploads/2019/08/IDLock150_ZWave_UserManual_v3.02.pdf) and on the home-assistant site [https://www.home-assistant.io/docs/z-wave/adding/](https://www.home-assistant.io/docs/z-wave/adding/).

To remove ID Lock 150 from Home Assistant:
* Go to the Z-Wave control panel in the Home Assistant frontend
* Click the Remove Node button in the Z-Wave Network Management card - this will place the controller in exclusion mode
* Open the door, press and hold the key-button three seconds. Use the mastercode followed by an `*`. 
* Press `2` followed by a `*` for all settings.
* Then press `0` to reset all settings on the Z-Wave module
* Wait for a sound and blue led for confirmation of the setting. 
* Run a Heal Network so all the other nodes learn about its removal
* The device will now be removed, but that won’t show until you restart Home Assistant
* Restart Home Assistant and wait for it to be fired all the way up
* Do a Heal Network again, just for kicsk
* Remove the id_lock150 entities from Entities
* Restart Home Assistant, ID Lock 150 should be all gone.

Somehow, this did a full reset on my ID Lock. I had to configure all the keys and stuffs, hinge and whatnot before I did the following:

To add ID Lock 150 to Home Assistant:
* Go to the Z-Wave control panel in the Home Assistant frontend
* Click the Add Node Secure button in the Z-Wave Network Management card - this will place the controller in inclusion mode
* Open the door, press and hold the key-button three seconds. Use the mastercode followed by an `*`. 
* Press `2` followed by a `*` for all settings.
* Then press `5` to set the lock in inclusion mode
* Wait for a sound and blue led for confirmation of the setting. 
* With the device in its final location, run a Heal Network

And now I had two instances of the ID Lock 150 in Home Assistant. I had to follow the guide "Removing Devices" with `is_failed: true` setting: [https://www.home-assistant.io/docs/z-wave/adding/](https://www.home-assistant.io/docs/z-wave/adding/).


## Legacy
Older notes which I have kept. 

### Remote access with TLS/SSL via Let's Encrypt
I am not using this anymore. I am using nginx reverse proxy. 
Follow this excellent guide over here [https://www.home-assistant.io/docs/ecosystem/certificates/lets_encrypt/](https://www.home-assistant.io/docs/ecosystem/certificates/lets_encrypt/)

You have to add the user `homeassistant` to the `etc/sudoers` file, so it can run the `certbot-auto` with nopassword.

Adding the following line to the `etc/sudoers` at the bottom with `sudo visudo`:
```bash
assistant@linuxbabe:/home/homeassistant/.homeassistant$ sudo visudo
(...)
homeassistant ALL=(ALL) NOPASSWD:SETENV: /home/homeassistant/certbot/certbot-auto
```
In this way you, only allow the user/systemuser that is running the homeassistant process (`hass`) only to run the cerbot for generating and renewing the certificate. This should be more secure in case of a security breach within the process `hass`.

### Apple Home dependencies
Controlling Home Assistant entities from Apple Home app on your iOs. 
Install the necessary dependencies to be able to forward entities from Home Assistant to Apple HomeKit, so they can be controlled from Apple’s Home app and Siri.
```bash
assistant@linuxbabe:/home/homeassistant/.homeassistant$ sudo apt-get install libavahi-compat-libdnssd-dev
```

Add `homekit:` in `configuration.yaml`:
```bash
assistant@linuxbabe:/home/homeassistant/.homeassistant$ sudo nano configuration.yaml 
# Enable Apple Home
homekit:
  auto_start: false
```
Depending on your setup, it might be necessary to disable Auto Start for all accessories to be available for HomeKit. Only those entities that are fully set up when the HomeKit integration is started, can be added. To start HomeKit when `auto_start: false`, you can call the service `homekit.start`.

If you have Z-Wave entities you want to be exposed to HomeKit, then you’ll need to disable auto start and then start it after the Z-Wave mesh is ready. This is because the Z-Wave entities won’t be fully set up until then. This can be automated using an automation. Let us edit `automations.yaml` and make Home Assistant wait 9 minutes before starting `homekit`:

```bash
assistant@linuxbabe:/home/homeassistant/.homeassistant$ sudo nano automations.yaml 
- alias: 'Start HomeKit'
  trigger:
  - platform: homeassistant
    event: start
  action:
  - delay: 00:09
  - service: homekit.start
```

Restart Home Assistant and you should be able to add it as a accessory in Apple Home on your iOS device. 
PS: Remember to properly configure your Apple TV as a hub, first. 

### Restrict number of devices
Perhaps you would not like to send everything from Home Assistant to your iOS device. Edit `configuration.yaml` and use a `filter`:
```bash

homekit:
  auto_start: False
  filter:
    include_entities:
      - binary_sensor.aeon_labs_zw100_multisensor_6_sensor
      - sensor.aeon_labs_zw100_multisensor_6_battery_level
      - sensor.aeon_labs_zw100_multisensor_6_relative_humidity
      - sensor.aeon_labs_zw100_multisensor_6_temperature
      - sensor.aeon_labs_zw100_multisensor_6_burglar
      - sensor.fibaro_system_fgbs001_universal_binary_sensor_temperature
      - sensor.fibaro_system_fgbs001_universal_binary_sensor_temperature_2
      - switch.fibaro_system_fgwpe_f_wall_plug_gen5_switch
      - lock.id_lock_as_150_locked
      - binary_sensor.burglar_up

```

### Upgrade python3.6 to python3.7
If you are upgrading Python from another version, you'll have to re-build your virtual environment.

Follow this guide [https://www.itsupportwale.com/blog/how-to-upgrade-to-python-3-7-on-ubuntu-18-10/](https://www.itsupportwale.com/blog/how-to-upgrade-to-python-3-7-on-ubuntu-18-10/) to update Python:
```bash
assistant@linuxbabe:/srv$ python3 -V
Python 3.6.8
assistant@linuxbabe:/srv$ sudo apt-get install python3.7 
assistant@linuxbabe:/srv$ sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.6 1
update-alternatives: using /usr/bin/python3.6 to provide /usr/bin/python3 (python3) in auto mode
assistant@linuxbabe:/srv$ sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.7 2
update-alternatives: using /usr/bin/python3.7 to provide /usr/bin/python3 (python3) in auto mode
assistant@linuxbabe:/srv$ sudo update-alternatives --config python3
There are 2 choices for the alternative python3 (providing /usr/bin/python3).

  Selection    Path                Priority   Status
------------------------------------------------------------
* 0            /usr/bin/python3.7   2         auto mode
  1            /usr/bin/python3.6   1         manual mode
  2            /usr/bin/python3.7   2         manual mode

Press <enter> to keep the current choice[*], or type selection number: 2
assistant@linuxbabe:/srv$ 
assistant@linuxbabe:/srv$ python3 -V
Python 3.7.5
assistant@linuxbabe:/srv$ sudo apt-get install python3.7-venv
```

If you’ve upgraded Python (for example, you were running 3.7.1 and now you’ve installed 3.7.3) then you’ll need to build a new virtual environment. Simply rename your existing virtual environment directory and follow the install steps again:
```bash
assistant@linuxbabe:/srv$ sudo mv homeassistant homeassistant.old
assistant@linuxbabe:/srv$ sudo mkdir homeassistant
assistant@linuxbabe:/srv$ sudo chown homeassistant:homeassistant homeassistant
assistant@linuxbabe:/srv$ sudo -u homeassistant -H -s
homeassistant@linuxbabe:/srv$ cd /srv/homeassistant
homeassistant@linuxbabe:/srv/homeassistant$ python3 -m venv .
homeassistant@linuxbabe:/srv/homeassistant$ source bin/activate
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ python3 -m pip install wheel
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ pip3 install homeassistant
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ hass
```

### Z-Wave Stick
We want to configure this device. Add this above `group: !include groups.yaml` in the `configuration.yaml` file:

```bash
assistant@linuxbabe:~$ cd /home/homeassistant/.homeassistant/
assistant@linuxbabe:/home/homeassistant/.homeassistant$ sudo nano configuration.yaml 
# Aeotec Z-Stick Gen5
zwave:
  usb_path: /dev/ttyACM0
  network_key: !secret network_key

```

Include the secret network key, read more here about the network key management: [https://www.home-assistant.io/docs/z-wave/installation/](https://www.home-assistant.io/docs/z-wave/installation/)
```bash
assistant@linuxbabe:~$ cd /home/homeassistant/.homeassistant/
assistant@linuxbabe:/home/homeassistant/.homeassistant$ sudo nano secrets.yaml 

# Use this file to store secrets like usernames and passwords.
# Learn more at https://home-assistant.io/docs/configuration/secrets/
some_password: welcome
network_key: "0x2D, 0xE2, 0x1F, 0xAB, 0xAE, 0xA8, 0xF2, 0xXE, 0x79, 0x2D, 0x3C, 0x21, 0x11, 0x3E, 0xF5, 0x9E"
```
Secure the `secrets.yaml` file:
```bash
assistant@linuxbabe:/home/homeassistant/.homeassistant$ sudo chmod 600 secrets.yaml 
```
If this unit has been paired with Z-Wave devices from before, the devices will be added automatically in Home Assistant. 

### Z-Wave dependencies
Install the necesseary dependencies to make Z-Wave work:
```bash
$ sudo apt-get install -y make libudev-dev g++ libyaml-dev libdpkg-perl
```

Restart Home Assistant through Configuration > Server Controls > and Server Management or with `sudo systemctl restart home-assistant@homeassistant`.

The first time Home Assistant uses `zwave` it will download `python_openzwave` in the background (Z-Wave drivers). It might take some time- 5 minutes? 
Wait until `options.xml` shows up in your directory. 

Edit the newly downloaded `options.xml` file and add the same network key there as well:
```bash
assistant@linuxbabe:/home/homeassistant/.homeassistant$ sudo nano options.xml 
(...)
<Option name="NetworkKey" value="0x2D, 0xE2, 0x1F, 0xAB, 0xAE, 0xA8, 0xF2, 0xXE, 0x79, 0x2D, 0x3C, 0x21, 0x11, 0x3E, 0xF5, 0x9E" />
(...)
```
As we did with our `secrets.yaml` file, secure the `options.xml` too:
```bash
assistant@linuxbabe:/home/homeassistant/.homeassistant$ sudo chmod 600 options.xml 
```
Restart Home Assistant again;
```bash
assistant@linuxbabe:/home/homeassistant/.homeassistant$ sudo systemctl restart home-assistant@homeassistant
```

## Authors
Mr. Johnson


## Acknowledgments
* [https://community.letsencrypt.org/t/certbot-command-not-found/16547/13](https://community.letsencrypt.org/t/certbot-command-not-found/16547/13)
* [https://community.home-assistant.io/t/configuring-z-wave-door-sensor/107408/6?u=gsemet](https://community.home-assistant.io/t/configuring-z-wave-door-sensor/107408/6?u=gsemet)
* [https://translate.google.com/translate?sl=auto&tl=en&u=https%3A%2F%2Fforum.jeedom.com%2Fviewtopic.php%3Ft%3D39395](https://translate.google.com/translate?sl=auto&tl=en&u=https%3A%2F%2Fforum.jeedom.com%2Fviewtopic.php%3Ft%3D39395)
* [https://www.home-assistant.io/integrations/zha/](https://www.home-assistant.io/integrations/zha/)
* [https://josiahvorst.com/a-superior-zigbee-experience-with-deconz-conbee-ii-home-assistant/](https://josiahvorst.com/a-superior-zigbee-experience-with-deconz-conbee-ii-home-assistant/)
* [https://www.home-assistant.io/integrations/systemmonitor](https://www.home-assistant.io/integrations/systemmonitor)
* [https://community.home-assistant.io/t/how-to-add-pic-to-device-tracker-device/14718/13](https://community.home-assistant.io/t/how-to-add-pic-to-device-tracker-device/14718/13)
* [https://www.home-assistant.io/integrations/ping](https://www.home-assistant.io/integrations/ping)
* [https://idlock.no/wp-content/uploads/2019/08/IDLock150_ZWave_UserManual_v3.02.pdf](https://idlock.no/wp-content/uploads/2019/08/IDLock150_ZWave_UserManual_v3.02.pdf)
* [https://community.home-assistant.io/t/z-wave-polling-questions-answered/2394/59](https://community.home-assistant.io/t/z-wave-polling-questions-answered/2394/59)
* [https://community.home-assistant.io/t/aeotec-sensors-are-not-triggering-state-changes/60042](https://community.home-assistant.io/t/aeotec-sensors-are-not-triggering-state-changes/60042)
* [https://www.home-assistant.io/integrations/binary_sensor.template/](https://www.home-assistant.io/integrations/binary_sensor.template/)
* [https://www.home-assistant.io/integrations/homekit/](https://www.home-assistant.io/integrations/homekit/)
* [https://blog.ceard.tech/2017/12/installing-home-assistant-in-virtual.html](https://blog.ceard.tech/2017/12/installing-home-assistant-in-virtual.html)
* [https://www.home-assistant.io/components/logger/](https://www.home-assistant.io/components/logger/)
* [https://aeotec.freshdesk.com/support/solutions/articles/6000121947-how-to-manually-factory-reset-z-stick-gen5-](https://aeotec.freshdesk.com/support/solutions/articles/6000121947-how-to-manually-factory-reset-z-stick-gen5-)
* [https://help.ubuntu.com/community/How%20to%20Create%20a%20Network%20Share%20Via%20Samba%20Via%20CLI%20(Command-line%20interface/Linux%20Terminal)%20-%20Uncomplicated,%20Simple%20and%20Brief%20Way!](https://help.ubuntu.com/community/How%20to%20Create%20a%20Network%20Share%20Via%20Samba%20Via%20CLI%20(Command-line%20interface/Linux%20Terminal)%20-%20Uncomplicated,%20Simple%20and%20Brief%20Way!)
* [https://community.home-assistant.io/t/integration-with-id-lock-150/78645/30](https://community.home-assistant.io/t/integration-with-id-lock-150/78645/30)
* [https://community.home-assistant.io/t/setting-up-openzwave-from-0-110/197709/13](https://community.home-assistant.io/t/setting-up-openzwave-from-0-110/197709/13)
* [https://community.home-assistant.io/t/so-when-is-openzwave-ozw-version-1-6-coming-to-home-assistant/127731/44](https://community.home-assistant.io/t/so-when-is-openzwave-ozw-version-1-6-coming-to-home-assistant/127731/44)
* [https://github.com/OpenZWave/Zwave2Mqtt/issues/413](https://github.com/OpenZWave/Zwave2Mqtt/issues/413)
* [https://linuxize.com/post/how-to-list-docker-containers/](https://linuxize.com/post/how-to-list-docker-containers/)
* [https://www.reddit.com/r/portainer/comments/inyjvd/how_to_upgrade_to_20/](https://www.reddit.com/r/portainer/comments/inyjvd/how_to_upgrade_to_20/)
* [https://www.youtube.com/watch?v=Of1gpoKP2mQ](https://www.youtube.com/watch?v=Of1gpoKP2mQ)
* [https://phoenixnap.com/kb/how-to-list-start-stop-docker-containers](https://phoenixnap.com/kb/how-to-list-start-stop-docker-containers)
* [https://stackoverflow.com/questions/13914226/how-to-know-which-device-is-connected-in-which-dev-ttyusb-port](https://stackoverflow.com/questions/13914226/how-to-know-which-device-is-connected-in-which-dev-ttyusb-port)
* [https://gist.github.com/xbmcnut/4be84e447c28fa7bc03e3488d85bb744](https://gist.github.com/xbmcnut/4be84e447c28fa7bc03e3488d85bb744)

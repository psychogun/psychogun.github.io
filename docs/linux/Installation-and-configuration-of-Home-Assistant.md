---
layout: default
title: Installation and configuration of Home Assistant
parent: Linux
nav_order: 7
---
*WORK IN PROGRESS as of 2019-08-30*
# Installation and configuration of Home Assistant
{: .no_toc }
This is how I used Home Assistant to send me events through Apple's Home app, with the help of an Ubuntu installation allocated on a ~~VM in ESXi 6.7~~ Proxmox Virtual Environment 6.1-5 with USB devices passed through the host. 
I have some lights, a smartlock, some devices to measure temperature and, hm, what else, a motion detector that triggers the light if it is dark. This howto came to life when I decided to use Proxmox instead of ESXi. 

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}
---

## Getting started
This is my guide. There are many like it, but this one is mine. My guide is my best friend. It is my life. 
I must master it as I must master my life. Without me, my guide is useless. Without my guide I am useless. I must trigger my guide true. 

### Prerequsites

* Proxmox Virtual Environment 6.1-5
* Ubuntu 18.04.3 Server
* Home Assistant 0.104.3
* Deconz Docker

* Aeotec ZW090 Z-Stick Gen5 EU
* Aeotec ZW100 MultiSensor 6
* FIBARO System FGBS001 Universal Binary Sensor
* FIBARO System FGWPE/F Wall plug Gen5
* ID Lock AS ID Lock 150
* Philips Hue light
* IKEA TRADFRI light
* Apple TV 4th gen

~~## Allocating a VM in ESXi 6.7
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
* standard system utilities~~


## Installation of Home Assistant
I am using a virtual environment provided by Proxmox. Under the creation of the guest host, remember to install OpenSSH Server.
```bash
sudo apt-get update
sudo apt-get upgrade
assistant@linuxbabe:~$ sudo apt-get install qemu-guest-agent
```
Set the correct date and time:
```bash
assistant@linuxbabe:~$ sudo dpkg-reconfigure tzdata
```

`sudo shutdown now` the Guest host and enable QEMU Guest Agent under Options in the Proxmox web gui. 

## Upgrade Python
Which version do you currently have?
```bash
assistant@linuxbabe:~$ python3 -V
Python 3.6.9
```
Install `python3.8`:
```bash
assistant@linuxbabe:~$ sudo apt-get install python3.8
assistant@linuxbabe:~$ python3.8 -V
Python 3.8.0
```
Add python3.6 & python3.8 to update-alternatives:
```bash
assistant@linuxbabe:~$ sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.6 1
update-alternatives: using /usr/bin/python3.6 to provide /usr/bin/python3 (python3) in auto mode
assistant@linuxbabe:~$ sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.8 2
update-alternatives: using /usr/bin/python3.8 to provide /usr/bin/python3 (python3) in auto mode
```
Update python3 to point to python3.8:
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
Test the version of python:
```bash
assistant@linuxbabe:~$ python3 -V
Python 3.8.0
```
### Install the dependencies
Fire up the guest host again and log in and install all the required dependencies for running Home Assistant in an python virtual environment:
```bash
assistant@linuxbabe:~$ sudo apt-get install python3.8-dev python3.8-venv python3-pip libffi-dev libssl-dev libudev-dev build-essential
```

Add an account for Home Assistant called `homeassistant`. Since this account is only for running Home Assistant the extra arguments of `-rm` is added to create a system account and create a home directory. The arguments `-G dialout` adds the user to the `dialout` group. The `dialout` group is required for using Z-Wave and Zigbee controllers.

```bash
assistant@linuxbabe:~$ sudo useradd -rm homeassistant -G dialout
```
Next we will create a directory for the installation of Home Assistant and change the owner to the homeassistant account.

```bash
assistant@linuxbabe:~$ cd /srv
assistant@linuxbabe:/srv$ sudo mkdir homeassistant
assistant@linuxbabe:/srv$ sudo chown homeassistant:homeassistant homeassistant
```

Next up is to create and change to a virtual environment for Home Assistant. This will be done as the homeassistant account.

```bash
assistant@linuxbabe:/srv$ sudo -u homeassistant -H -s
homeassistant@linuxbabe:/srv$ cd /srv/homeassistant
homeassistant@linuxbabe:/srv/homeassistant$ python3 -m venv .
homeassistant@linuxbabe:/srv/homeassistant$ source bin/activate
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ 
```

Once you have activated the virtual environment (notice the prompt change to `(homeassistant) homeassistant@raspberrypi:/srv/homeassistant $)` you will need to run the following command to install a required python package.

```bash
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ python3 -m pip install wheel
Collecting wheel
  Downloading https://files.pythonhosted.org/packages/00/83/b4a77d044e78ad1a45610eb88f745be2fd2c6d658f9798a15e384b7d57c9/wheel-0.33.6-py2.py3-none-any.whl
Installing collected packages: wheel
Successfully installed wheel-0.33.6
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ 
```

Once you have installed the required python package it is now time to install Home Assistant!

### Install Home Assistant

```bash
homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ pip3 install homeassistant
```

Start Home Assistant for the first time. This will complete the installation for you, automatically creating the `.homeassistant` configuration directory in the `/home/homeassistant` directory, and installing any basic dependencies.

```bash
(homeassistant) homeassistant@linuxbabe:/srv/homeassistant$ hass
```

When you run the hass command for the first time, it will download, install and cache the necessary libraries/dependencies. This procedure may take anywhere between 5 to 10 minutes. During that time, you may get “site cannot be reached” error when accessing the web interface. This will only happen for the first time, and subsequent restarts will be much faster.

You can now reach your installation on your Raspberry Pi over the web interface on `http://ipaddress:8123`. Hop on in and configure user and set your timezone and location. 

Further more, you are able to set up integrations from the web service, but that did not work out, at least for me when I tried to install Z-Wave, it just hangs on `2020-01-29 16:50:52 INFO (SyncWorker_2) [homeassistant.util.package] Attempting install of homeassistant-pyozw==0.1.7` ..

I takes a while, but then you are able to configure Z-Wave:
USB Path: /dev/ttyACM0
Network Key: 

### Home Assistant as a daemon
Do this if you would like Home Assistant to start on boot.
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
You need to reload systemd to make the daemon aware of the new configuration.
```bash
assistant@linuxbabe:/srv$ sudo systemctl --system daemon-reload
```
To have Home Assistant start automatically at boot, enable the service.
```bash
assistant@linuxbabe:/srv$ sudo systemctl enable home-assistant@homeassistant
Created symlink /etc/systemd/system/multi-user.target.wants/home-assistant@homeassistant.service → /etc/systemd/system/home-assistant@homeassistant.service.
assistant@linuxbabe:/srv$ 
```
To disable the automatic start, use this command.
```bash
assistant@linuxbabe:/srv$ sudo systemctl disable home-assistant@homeassistant
Removed /etc/systemd/system/multi-user.target.wants/home-assistant@homeassistant.service.
```

To start Home Assistant now, use this command.
```bash
assistant@linuxbabe:/srv$ sudo systemctl start home-assistant@homeassistant
```

You can also substitute the `start` above with `stop` to stop Home Assistant, `restart` to restart Home Assistant, and `status` to see a brief status report as seen below.
```bash
sudo systemctl enable home-assistant@homeassistant
```

To get Home Assistant’s logging output, simple use journalctl.
```bash
$ sudo journalctl -f -u home-assistant@homeassistant
```
Because the log can scroll quite quickly, you can select to view only the error lines:
```bash
$ sudo journalctl -f -u home-assistant@homeasisstant | grep -i 'error'
```
When working on Home Assistant, you can easily restart the system and then watch the log output by combining the above commands using &&
```
$ sudo systemctl restart home-assistant@YOUR_USER && sudo journalctl -f -u home-assistant@YOUR_USER
```

Do these things that are stated above, and shut down your VM guest through Proxmox web gui.

## Attaching Z-Wave controllers
I am using a Aeotec ZW090 Z-Stick Gen5 EU for communication with my ID Lock 150. We will have to pass USB device through the host to the guest OS. 

First, list available USB devices on the proxmox host:
```bash
root@h37bproxmox01:~# lsusb 
Bus 002 Device 002: ID 8087:8000 Intel Corp. 
Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 001 Device 002: ID 8087:8008 Intel Corp. 
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 004 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
root@h37bproxmox01:~# 
```

Plug in the Z-Stick USB dongle:
```bash
root@h37bproxmox01:~# lsusb
Bus 002 Device 002: ID 8087:8000 Intel Corp. 
Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 001 Device 002: ID 8087:8008 Intel Corp. 
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 004 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 003 Device 002: ID 0658:0200 Sigma Designs, Inc. Aeotec Z-Stick Gen5 (ZW090) - UZB
Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
root@h37bproxmox01:~# 
```
All right, it is attached with and `ID` of `0658:0200`. 

It does not automatically attach itself in our Virtual Machine, as you can see:
```bash
assistant@linuxbabe:~$ lsusb 
Bus 001 Device 002: ID 0627:0001 Adomax Technology Co., Ltd 
Bus 001 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
```
Select the Home Assistant Virtual Machine in Proxmox and go to Hardware. Add > USB Device. Select `* Use USB Vendor/Device ID` and the device with the above ID (`Unknown (0658:0200)`).
Reboot the Virtual Machine. 

Now it is attached:
```bash
assistant@linuxbabe:~$ lsusb 
Bus 003 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 002 Device 002: ID 0658:0200 Sigma Designs, Inc. 
Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 001 Device 002: ID 0627:0001 Adomax Technology Co., Ltd 
Bus 001 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
```
It is configured as `/dev/ttyACM0`.

We want to configure this device. Add this above `group: !include groups.yaml` in the `configuration.yaml` file:

```bash
assistant@linuxbabe:~$ cd /home/homeassistant/.homeassistant/
assistant@linuxbabe:/home/homeassistant/.homeassistant$ sudo nano configuration.yaml 
# Aeotec Z-Stick Gen5
zwave:
  usb_path: /dev/ttyACM0
  network_key: !secret network_key

```
If this unit has been paired with Z-Wave devices from before, they will be added automatically in Home Assistant. 

```bash
assistant@linuxbabe:~$ cd /home/homeassistant/.homeassistant/
assistant@linuxbabe:/home/homeassistant/.homeassistant$ sudo nano secrets.yaml 

# Use this file to store secrets like usernames and passwords.
# Learn more at https://home-assistant.io/docs/configuration/secrets/
some_password: welcome
network_key: "0x2D, 0xE2, 0x1F, 0xAB, 0xAE, 0xA8, 0xF2, 0xXE, 0x79, 0x2D, 0x3C, 0x21, 0x11, 0x3E, 0xF5, 0x9E"

assistant@linuxbabe:/home/homeassistant/.homeassistant$ sudo chmod 600 secrets.yaml 
```

(Remember to also add `network_key:` underneath `usb_path:` if this has been generated).

### Z-Wave dependencies
Install the necesseary dependencies to make Z-Wave work:
```bash
$ sudo apt-get install -y make libudev-dev g++ libyaml-dev libdpkg-perl python3.7-dev
```

Restart Home Assistant through Configuration > Server Controls > and Server Management. 

The first time Home Assistant uses `zwave` it will download `python_openzwave` in the background (Z-Wave drivers). It might take some time. 


## Controlling Home Assistant from Apple Home
### Apple Home dependencies
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
## Advanced Mode
In the Home Assistant GUI, go to your profile by clicking your name down to the left. Enable `Advanced Mode`. 

With Advanced Mode turned on, you can go to `Developer Tools` and check your 

## System Health
Enable System Health by adding `system_health:` in `configuration.yaml`, add it to the very top:

```bash
assistant@linuxbabe:/home/homeassistant/.homeassistant$ sudo nano configuration.yaml 
# Enable System Health
system_health:
```

## Adding a tampering sensor
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

Then go to Configuration > Z-Wave Node Management 
Select

Nodes: AEON Labs ZW100 Multisensor 6

Entities of this node:
binary_sensor.aeon_labs_zw100_multisensor_6_sensor

Node Values: Burglag (Instance; 1, Index; 10)

Node group associations: 

Node Config Options: 
3600

Config Parameter: 5; Command Options

Config Value: ** Binary Sensor Report **

Click `SET CONFIG PARAMETER`.

* Tap the Action Button on Multisensor 6 to put Multisensor 6 back to sleep, or wait 10 minutes. (recommended to manually put it back to sleep to conserve battery life).

* Make sure that also the entity `binary_sensor.aeon_labs_zw100_multisensor_6_sensor` also uses `Binary Sensor Report`, as stated as best practise here: [https://www.home-assistant.io/docs/z-wave/entities/](https://www.home-assistant.io/docs/z-wave/entities/).


## Remote access with TLS/SSL via Let's Encrypt
Follow this excellent guide over here [https://www.home-assistant.io/docs/ecosystem/certificates/lets_encrypt/](https://www.home-assistant.io/docs/ecosystem/certificates/lets_encrypt/)

## Change icons
Change your icons by using `mdi:icon` - look at this list for reference: [https://cdn.materialdesignicons.com/3.2.89/](https://cdn.materialdesignicons.com/3.2.89/)


## Upgrade python3.6 to python3.7
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

## Backup Home Assistant



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

## Fault finding
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

#### ID Lock 150 configuration
Go to Configuration > Z-Wave > Select Node `ID Lock AS 150 (Node 7: Complete)
* Node Config Options: 3: Door Hinge Position
* Config Value: Left Handle
Click `SET CONFIG PARAMETER`.

* Node Config Options: 4: Door Audio Volume Level
* Config Value: No Sound
Click `SET CONFIG PARAMETER`.


## Ping
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


```bash

# Example configuration.yaml entry
sensor:
  - platform: systemmonitor
    resources:
      - type: disk_use_percent
        arg: /home
      - type: memory_free

```


To use a picture from facebook 10159577526160105
https://graph.facebook.com/10159577526160105/picture?type=normal

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
PS: `~` is the current user's home directory.

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

Restart the samba:
```bash
assistant@linuxbabe:~$ sudo service smbd restart
```
Once Samba has restarted, use this command to check your smb.conf for any syntax errors
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

On MacOS, open Finder and press `command + K`. Connect to server `smb://172.16.32.17/homeassistant/.homeassistant` to see your configuration files. Now you can directly edit the files from there.

## Enable Multi-factor Authentication Modules
Go to your profile by clicking your name down to the left.

Under `Multi-factor Authentication Modules`, click ENABLE on `totp`. Download Google Authenticator from your App Store and scan the code. For more information, go to [https://www.home-assistant.io/docs/authentication/multi-factor-auth/](https://www.home-assistant.io/docs/authentication/multi-factor-auth/).

### Home Assistant Companion
Install Home Assistant Companion from your App Store, but only do it after you have enabled Multi-factor. Do not log in and use it before enabling multi-factor authentication, because you will be unable to log on in without destroying your entities (because you are unable to connect, because you have not used Google Authenticator under that process).

## Zigbee

There are different ways to connect your Zigbee devices to Home Assistant (Zigbee2mqtt, deCONZ, ZHA). I opted for using ZHA (the integrated one). 

Shut the proxmox guest from proxmox. Attach the USB physically, add it to the guest and start home assistant again. My device showed up as `FT230X Basic UART (0403:6015)` in proxmox. 

```bash
assistant@linuxbabe:~$ lsusb 
Bus 002 Device 003: ID 0403:6015 Future Technology Devices International, Ltd Bridge(I2C/SPI/UART/FIFO)
assistant@linuxbabe:~$ ls -alh /dev/ttyUSB0 
crw-rw---- 1 root dialout 188, 0 Feb  4 22:39 /dev/ttyUSB0
```
Following this guide [https://www.home-assistant.io/integrations/zha/](https://www.home-assistant.io/integrations/zha/)

```
ZHA
USB Device Path: /dev/ttyUSB0
Radio Type: deconz
```


deCONZ by dresden elektronik is a software that communicates with ConBee/RaspBee Zigbee gateways and exposes Zigbee devices that are connected to the gateway.



## Authors
Mr. Johnson

## Acknowledgments
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
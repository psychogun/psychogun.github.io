---
layout: default
title: USB-passthrough
parent: xcp-ng
nav_order: 1
---
# USB-passthrough
{: .no_toc }
This is how I passed through a Z-Wave USB dongle to a Home Assistant Operating System guest. Just some personal notes, it is basically this guide [https://xcp-ng.org/docs/compute.html#usb-passthrough](https://xcp-ng.org/docs/compute.html#usb-passthrough).

I have not found out if this survives a reboot of the host. Probably will not. I would not recommend this, as adding USB devices in Proxmox is very trivial compared to xcp-ng. 

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
I am using an Home Assistant installation on a virtual machine deployed from XCP-NG. I do not remember how I installed it, I think somehow somewhere along the line I used someones script to convert the HA machine to a usable format for xcp-ng. 

### Prerequsites
* Home Assistant OS 6.1
* xcp-ng 8.2

---

## Attaching USB device
List USB devices  on the host:
```bash
[16:03 xcp-ng ~]# lsusb
Bus 004 Device 002: ID 8087:8000 Intel Corp. 
Bus 004 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 002 Device 002: ID 8087:8008 Intel Corp. 
Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 003 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

Plug in the Z-Wave USB device:
```bash
[16:03 xcp-ng ~]# lsusb
Bus 004 Device 002: ID 8087:8000 Intel Corp. 
Bus 004 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 001 Device 002: ID 8087:8008 Intel Corp. 
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 003 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 002 Device 002: ID 0658:0200 Sigma Designs, Inc. Aeotec Z-Stick Gen5 (ZW090) - UZB
Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
[16:06 xcp-ng ~]# 
```

Found it:
```bash
Bus 002 Device 002: ID 0658:0200 Sigma Designs, Inc. Aeotec Z-Stick Gen5 (ZW090) - UZB
```

Let us find  the `uuid` of the Z-Wave USB dongle::
```bash
[16:06 xcp-ng ~]# xe pusb-list 
uuid ( RO)            : a4650827-f53a-33d9-b1ac-e22c06440f09
            path ( RO): 1-10
       vendor-id ( RO): 0403
     vendor-desc ( RO): Future Technology Devices International, Ltd
      product-id ( RO): 6015
    product-desc ( RO): Bridge(I2C/SPI/UART/FIFO)
          serial ( RO): DO00K1Q1
         version ( RO): 2.00
     description ( RO): Future Technology Devices International, Ltd_Bridge(I2C/SPI/UART/FIFO)_DO00K1Q1
           speed ( RO): 12.000

```
If nothing shows in the `pusb-list` command above, try this:

Run a more verbose `lsusb` on this device:
```bash
[18:49 xcp-ng ~]# lsusb -v -d 0658:0200

Bus 002 Device 002: ID 0658:0200 Sigma Designs, Inc. Aeotec Z-Stick Gen5 (ZW090) - UZB
Device Descriptor:
  bLength                18
  bDescriptorType         1
  bcdUSB               2.00
  bDeviceClass            2 Communications
  bDeviceSubClass         0 
  bDeviceProtocol         0 
  bMaxPacketSize0         8
  idVendor           0x0658 Sigma Designs, Inc.
  idProduct          0x0200 Aeotec Z-Stick Gen5 (ZW090) - UZB
  bcdDevice            0.00
  iManufacturer           0 
  iProduct                0 
  iSerial                 0 
  bNumConfigurations      1
  Configuration Descriptor:
    bLength                 9
    bDescriptorType         2
    wTotalLength           67
    bNumInterfaces          2
    bConfigurationValue     1
    iConfiguration          0 
    bmAttributes         0x80
      (Bus Powered)
    MaxPower              100mA
    Interface Descriptor:
      bLength                 9
      bDescriptorType         4
      bInterfaceNumber        0
      bAlternateSetting       0
      bNumEndpoints           1
      bInterfaceClass         2 Communications
      bInterfaceSubClass      2 Abstract (modem)
      bInterfaceProtocol      1 AT-commands (v.25ter)
      iInterface              0 
      CDC Header:
        bcdCDC               1.10
      CDC Call Management:
        bmCapabilities       0x00
        bDataInterface          1
      CDC ACM:
        bmCapabilities       0x00
      CDC Union:
        bMasterInterface        0
        bSlaveInterface         1 
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x81  EP 1 IN
        bmAttributes            3
          Transfer Type            Interrupt
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0008  1x 8 bytes
        bInterval              32
    Interface Descriptor:
      bLength                 9
      bDescriptorType         4
      bInterfaceNumber        1
      bAlternateSetting       0
      bNumEndpoints           2
      bInterfaceClass        10 CDC Data
      bInterfaceSubClass      0 Unused
      bInterfaceProtocol      0 
      iInterface              0 
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x82  EP 2 IN
        bmAttributes            2
          Transfer Type            Bulk
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0020  1x 32 bytes
        bInterval               0
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x02  EP 2 OUT
        bmAttributes            2
          Transfer Type            Bulk
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0020  1x 32 bytes
        bInterval               0
Device Status:     0x0000
  (Bus Powered)

[18:50 xcp-ng ~]# 
```


Run `xe pusb-scan`to confirm the file can be parsed correctly:
```bash
[18:53 xcp-ng ~]# xe pusb-scan host-uuid=<host-uuid>
```

Let's enable this device for passthrough:
```bash
[21:02 xcp-ng ~]# xe pusb-param-set uuid=9f1a649b-15b6-a67b-584b-ae5639e97e84 passthrough-enabled=true
```

Let us find `group uuid`:
```bash
[21:03 xcp-ng ~]# xe usb-group-list PUSB-uuids=9f1a649b-15b6-a67b-584b-ae5639e97e84
uuid ( RO)                : 6d7c0694-6d96-f957-c1f3-4b124a8c9f8d
          name-label ( RW): Group of 0658 0200 USBs
    name-description ( RW): 


[16:26 xcp-ng ~]# 
```

Note  the `group uuid`: `6d7c0694-6d96-f957-c1f3-4b124a8c9f8d`.

List the current VMs:
```bash
[16:15 xcp-ng ~]# xe vm-list
uuid ( RO)           : cd7c086b-9216-1f42-f5fd-f6a9dbd967af
     name-label ( RW): homeassistant
    power-state ( RO): running


[16:18 xcp-ng ~]# 
```

Shut down the VM if it is running, as hot-plug is not supported (`xe vm-shutdown uuid=cd7c086b-9216-1f42-f5fd-f6a9dbd967af`):
```bash
[16:15 xcp-ng ~]# xe vm-list
uuid ( RO)           : cd7c086b-9216-1f42-f5fd-f6a9dbd967af
     name-label ( RW): homeassistant
    power-state ( RO): halted


[16:18 xcp-ng ~]# 
```

Attach the USB to the guest:
```bash
[21:04 xcp-ng ~]# xe vusb-create usb-group-uuid=6d7c0694-6d96-f957-c1f3-4b124a8c9f8d vm-uuid=cd7c086b-9216-1f42-f5fd-f6a9dbd967af 
562af132-ed12-fc22-9154-255279ba8d93
[16:29 xcp-ng ~]# 
```

Now you  can try to start the host:
```bash
[16:29 xcp-ng ~]# xe vm-start uuid=cd7c086b-9216-1f42-f5fd-f6a9dbd967af 
```

---

## Troubleshooting
### Verify attachment of USB device
Enable advanced mode in Home Assistant Operating System to be able to install the `Terminal & SSH` add-on. 
```bash

| |  | |                          /\           (_)   | |            | |  
| |__| | ___  _ __ ___   ___     /  \   ___ ___ _ ___| |_ __ _ _ __ | |_ 
|  __  |/ _ \| '_ \ _ \ / _ \   / /\ \ / __/ __| / __| __/ _\ | '_ \| __|
| |  | | (_) | | | | | |  __/  / ____ \\__ \__ \ \__ \ || (_| | | | | |_ 
|_|  |_|\___/|_| |_| |_|\___| /_/    \_\___/___/_|___/\__\__,_|_| |_|\__|

Welcome to the Home Assistant command line.

System information
  IPv4 addresses for eth0:  192.168.108.14/24

  OS Version:               Home Assistant OS 6.1
  Home Assistant Core:      2021.6.6

  Home Assistant URL:       http://homeassistant.local:8123
  Observer URL:             http://homeassistant.local:4357
```

Check out `lsusb`:
```bash
~ $ lsusb
Bus 001 Device 001: ID 1d6b:0001
Bus 001 Device 004: ID 0658:0200
Bus 001 Device 002: ID 0409:55aa
Bus 001 Device 003: ID 0627:0001
~ $ 
```
`Bus 001 Device 004: ID 0658:0200` is our device.

You'll probably find it again over at `/dev/serial/by-id`:
```bash
~ $ ls -alh /dev/serial/by-id/usb-0658_0200-if00 
lrwxrwxrwx    1 root     root          13 Jul  4 21:06 /dev/serial/by-id/usb-0658_0200-if00 -> ../../ttyACM0
~ $ 
```
As you can see, it is mapped to `/dev/ttyACM0`.

### Not supported
For the longest of time, I could not get the above to work. The problem was that I had attached the Zigbee USB dongle, and tried to use it as an Z-Wave controller. This, my friend, does not work at all. Remember to know which device you are working with :)

```bash
2021-07-04T18:40:23.200Z DRIVER   ███████╗ ██╗    ██╗  █████╗  ██╗   ██╗ ███████╗             ██╗ ███████╗
                                  ╚══███╔╝ ██║    ██║ ██╔══██╗ ██║   ██║ ██╔════╝             ██║ ██╔════╝
                                    ███╔╝  ██║ █╗ ██║ ███████║ ██║   ██║ █████╗   █████╗      ██║ ███████╗
                                   ███╔╝   ██║███╗██║ ██╔══██║ ╚██╗ ██╔╝ ██╔══╝   ╚════╝ ██   ██║ ╚════██║
                                  ███████╗ ╚███╔███╔╝ ██║  ██║  ╚████╔╝  ███████╗        ╚█████╔╝ ███████║
                                  ╚══════╝  ╚══╝╚══╝  ╚═╝  ╚═╝   ╚═══╝   ╚══════╝         ╚════╝  ╚══════╝
2021-07-04T18:40:23.204Z DRIVER   version 7.10.0
2021-07-04T18:40:23.204Z CONFIG   version 7.10.0
2021-07-04T18:40:23.205Z DRIVER   
2021-07-04T18:40:23.205Z DRIVER   starting driver...
2021-07-04T18:40:23.218Z DRIVER   opening serial port /dev/ttyUSB0
2021-07-04T18:40:23.302Z DRIVER   serial port opened
2021-07-04T18:40:24.807Z DRIVER   loading configuration...
2021-07-04T18:40:25.288Z DRIVER   beginning interview...
2021-07-04T18:40:25.290Z DRIVER   added request handler for AddNodeToNetwork (0x4a)...
                                  1 registered
2021-07-04T18:40:25.291Z DRIVER   added request handler for RemoveNodeFromNetwork (0x4b)...
                                  1 registered
2021-07-04T18:40:25.291Z DRIVER   added request handler for ReplaceFailedNode (0x63)...
                                  1 registered
2021-07-04T18:40:25.292Z CNTRLR   beginning interview...
2021-07-04T18:40:25.292Z CNTRLR   querying version info...
2021-07-04T18:40:25.370Z DRIVER » [REQ] [GetControllerVersion]
2021-07-04T18:40:26.373Z CNTRLR   Failed to execute controller command after 1/3 attempts. Scheduling next try i
                                  n 100 ms.
2021-07-04T18:40:26.474Z DRIVER » [REQ] [GetControllerVersion]
2021-07-04T18:40:27.477Z CNTRLR   Failed to execute controller command after 2/3 attempts. Scheduling next try i
                                  n 1100 ms.
2021-07-04T18:40:28.579Z DRIVER » [REQ] [GetControllerVersion]
2021-07-04T18:40:29.586Z DRIVER   Failed to initialize the driver: Timeout while waiting for an ACK from the con
                                  troller
Error in driver ZWaveError: Failed to initialize the driver: Timeout while waiting for an ACK from the controller
    at Immediate.<anonymous> (/usr/src/node_modules/zwave-js/src/lib/driver/Driver.ts:691:6) {
  code: 5,
  context: undefined,
  transactionSource: undefined
}
2021-07-04T18:40:29.591Z DRIVER   destroying driver instance...
[20:40:30] INFO: Successfully send discovery information to Home Assistant.
```

Moral of the story? Do not mistake the Conbee device for a Z-Wave device.

### usb-policy.conf
This is not required. Kept here for historical reasons. 

Add `ALLOW` entry on the very top of this list:
```bash
[18:51 xcp-ng ~]# nano /etc/xensource/usb-policy.conf 
# When you change this file, run 'xe pusb-scan' to confirm
# the file can be parsed correctly.
#
# Syntax is an ordered list of case insensitive rules where # is line comment
#  and each rule is (ALLOW | DENY) : ( match )*
#  and each match is (class|subclass|prot|vid|pid|rel) = hex-number
# Maximum hex value for class/subclass/prot is FF, and for vid/pid/rel is FFFF
#
# USB Hubs (class 09) are always denied, independently of the rules in this file
ALLOW:vid=0658 pid=0200 # Sigma Designs, Inc. Aeotec Z-Stick Gen5 (ZW090) - UZB
DENY: vid=17e9 # All DisplayLink USB displays
DENY: class=02 # Communications and CDC-Control
ALLOW:vid=056a pid=0315 class=03 # Wacom Intuos tablet
ALLOW:vid=056a pid=0314 class=03 # Wacom Intuos tablet
ALLOW:vid=056a pid=00fb class=03 # Wacom DTU tablet
DENY: class=03 subclass=01 prot=01 # HID Boot keyboards
DENY: class=03 subclass=01 prot=02 # HID Boot mice
DENY: class=0a # CDC-Data
DENY: class=0b # Smartcard
DENY: class=e0 # Wireless controller
DENY: class=ef subclass=04 # Miscellaneous network devices
ALLOW: # Otherwise allow everything else
```

---

## Authors
Mr. Johnson

---

## Acknowledgments
* [https://xcp-ng.org/docs/compute.html#usb-passthrough](https://xcp-ng.org/docs/compute.html#usb-passthrough)
* [https://github.com/xcp-ng/xcp/issues/125](https://github.com/xcp-ng/xcp/issues/125)
* [https://forums.lawrencesystems.com/t/xcp-ng-usb-passthrough/3190/3](https://forums.lawrencesystems.com/t/xcp-ng-usb-passthrough/3190/3)
* [https://xcp-ng.org/forum/topic/266/usb-passthrough-test-reports-in-7-5rc1/6](https://xcp-ng.org/forum/topic/266/usb-passthrough-test-reports-in-7-5rc1/6)
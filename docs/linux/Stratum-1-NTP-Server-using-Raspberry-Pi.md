---
layout: default
title: Stratum 1 NTP Server using Raspberry Pi
parent: Linux
nav_order: 15
---
# Stratum 1 NTP Server using Raspberry Pi
{: .no_toc }
This is how I used a GPS ~~USB dongle~~ HAT (Hardware Attached on Top) connected to a Raspberry Pi 3 to work stand-alone/and or with a secure (NTPSec) Internet server, to provide a NTP service on my network. The Raspberry Pi 3 now works as a Stratum 1 NTP server [https://timetoolsltd.com/ntp/what-is-a-stratum-1-time-server/](https://timetoolsltd.com/ntp/what-is-a-stratum-1-time-server/). 

I used this is conjunction with my PfSense firewall and should the GPS fail, I have configured a fallback of which the Raspberry Pi will pull time through `NTPSec`. NTPSec provides a secure method of retrieving time from public ntp servers with the help of TLS. 

~~Please realize that connecting a consumer-grade (designed for location, not for time keeping) GPS unit over USB serial does not have the accuracy of “proper” GPS synchronization due to significant and inconsistent delays in the serial line itself, as well as in the USB system. Your notion of time will likely be delayed by tens of milliseconds if not hundreds of milliseconds.~~ 

~~Any such-configured NTP server certainly can serve as a backup for local timekeeping when Internet connectivity is not available, but should never be advertised on the Internet as a Stratum 1 clock.~~

Proper GPS synchronization uses a PPS output (or similar) from the device, fed directly into an interrupt-generating line, for example a GPIO or parallel port. A GPSDO (GPS Disciplined Oscillator) is typically used which locks a temperature-compensated oscillator to the GPS time and provides a stable, reliable reference.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}
---
## Getting started


## Prerequisites
* Raspberry Pi Model 3
* ~~Globalsat BU-353S4 (`lsusb` shows `ID 067b:2303`)~~
* ~~Globalsat BU-353 USB GPS receiver originally used the SiRF Star III chipset, now also available with the SiRF Star IV chipset which offers enhanced performance (BU-353-S4 variant). Both versions use the Prolific PL2303 serial/USB chipset. This howto was done with the S4 variant. It is a high-quality device, readily available (Ebay etc.) for around $30.~~
* The GPS antenna will need to have a good view of GPS satellites 
* Adafruit Ultimate GPS HAT
* This was done from MacOS

## Installation instructions
I am using Raspbian Buster lite. 

I downloaded `2019-09-26-raspbian-buster-lite.img` from [https://www.raspberrypi.org/downloads/raspbian/](https://www.raspberrypi.org/downloads/raspbian/).

Follow the installation instructions provided here [https://www.raspberrypi.org/documentation/installation/installing-images/README.md](https://www.raspberrypi.org/documentation/installation/installing-images/README.md) to install the image/flash your SD card.

### Enable sshd
In order to remotely administer your headless Raspberry Pi server, you will need to enable the SSH server.

So how do you enable SSH server when Raspbian Buster Lite boots for the first time?

First, reinsert your microSD card to your MacOS powered device. When your computer loads up your microSD card, simply copy a blank file named as `ssh` into the root directory of your microSD card:
```zsh
zamba:~ don$ cd /Volumes/boot/
zamba:boot don$ touch ssh
```
Insert the microSDHC card to your Raspberry Pi, connect your network cable and finally your power supply. Go to your DHCP server and find out which IP adress the Raspberry Pi was given. Use SSH to connect, the default username and password is `pi` and `raspberry`. 

### Change password for user pi
```bash
zamba:~ don ssh -l pi 192.168.52.253
pi@raspberrypi:~ $ passwd
Changing password for pi.
Current password: 
New password: 
Retype new password: 
passwd: password updated successfully
```

### Change the pi username
```bash
root@raspberrypi:~# sudo passwd root
New password: 
Retype new password: 
passwd: password updated successfully
```
Enable `root` login:
```bash
pi@raspberrypi:~ $ sudo -i
root@raspberrypi:~# nano /etc/ssh/sshd_config 
(...)
PermitRootLogin yes
```
Restart the `ssh` service:
```bash
root@raspberrypi:~# /etc/init.d/ssh restart
[ ok ] Restarting ssh (via systemctl): ssh.service.
root@raspberrypi:~# 
```
Logout and log back in as the user `root`:
```bash
root@raspberrypi:~# logout
pi@raspberrypi:~ $ logout
Connection to 192.168.52.253 closed.
zamba:~ don ssh -l root 192.168.52.253
```
Use `usermod` to change user and reflect the new login name:
```bash
root@raspberrypi:~# usermod -l userntp pi
root@raspberrypi:~# usermod -m -d /home/userntp userntp
```
Disable `root` login:
```bash
root@raspberrypi:~# nano /etc/ssh/sshd_config
(...)
#PermitRootLogin yes
root@raspberrypi:~# /etc/init.d/ssh restart
root@raspberrypi:~# logout
Connection to 192.168.52.253 closed.
```
Log in using the new username `admin`:
```bash
zamba:~ don ssh -l admin 192.168.52.253
```
Does the user have `sudo` privileges?
```bash
admin@raspberrypi:~ $ sudo apt-get update
```
Disable the `root` account by locking the password:
```bash
admin@raspberrypi:~ $ sudo passwd -l root 
passwd: password expiry information changed.
```

### Change hostname
Use `raspi-config` and enter the `Network Options` menu and change the `Hostname` to something of your liking. This will require a reboot.

### Update & Upgrade
```bash
admin@raspberrypi:~ $ sudo apt-get update
admin@raspberrypi:~ $ sudo apt-get upgrade
admin@raspberrypi:~ $ sudo reboot
```

### Set your time zone
```bash
admin@raspberrypi:~ $ date
Sat 11 Jan 21:22:53 GMT 2020
admin@raspberrypi:~ $ sudo dpkg-reconfigure tzdata

Current default time zone: 'Europe/Paris'
Local time is now:      Sat Jan 11 22:24:07 CET 2020.
Universal Time is now:  Sat Jan 11 21:24:07 UTC 2020.

admin@raspberrypi:~ $ date
Sat 11 Jan 22:24:16 CET 2020
```

## Serial port
The Ultimate GPS hat delivers its data over a serial port at 9600 bps, which uses pins 14 and 15 on the GPIO header.
On the Raspberry Pi 2, this went directly to the hardware UART and was available to Linux on `/dev/ttyAMA0`. Assuming there was no serial console using it, GPS works great here.
But on the Pi 3, the high-performance hardware UART is used by the Bluetooth subsystem, and the serial port supported on GPIO pins 14+15 is emulated much more weakly and is available on `/dev/ttyS0`. 

This is pretty much a software UART, and I don't like it. 

Well, I do not know about that- most of these instructions were taken from this excellent guide [http://www.unixwiz.net/techtips/raspberry-pi3-gps-time.html](http://www.unixwiz.net/techtips/raspberry-pi3-gps-time.html).

```bash
/dev/ttyAMA0
```
- hardware UART for Bluetooth support

```bash
/dev/ttyS0
```
- software UART on GPIO 14+15

Fortunately, it's possible to shuffle around the hardware in software (!) to take back the hardware UART for our purposes.
### Disable the console getty programs
This refers to the serial console, which is generally useful, but as we prefer to use the UART for our GPS Hat, we have to disable the consoles. This is done in several steps.

```bash
admin@raspberrypi:~ $ sudo systemctl stop serial-getty@ttyAMA0.service
admin@raspberrypi:~ $ sudo systemctl disable serial-getty@ttyAMA0.service
```

This prevents the serial console programs from starting, but does not keep the Linux kernel from attempting to use it. This must be disabled by editing `/boot/cmdline.txt` and removing the serial portions shown in strikeout text (~~console=serial0,115200~~):
```bash
admin@raspberrypi:~ $ sudo nano /boot/cmdline.txt
admin@raspberrypi:~ $ more /boot/cmdline.txt 
console=serial0,115200 console=tty1 root=PARTUUID=6c586e13-02 rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait
admin@raspberrypi:~ $ 
```

Change to `console=tty1 root=PARTUUID=6c586e13-02 rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait`.

### Disable Bluetooth and steal the hardware UART
I didn't care about Bluetooth so was happy to disable it entirely and use the `/dev/ttyAMA0` serial port for the GPS, and this is easy to do with the Device Tree Overlay facility: these allow easy tailoring of low-level device behavior with a simple config file.
Edit `/boot/config.txt` and add the lines:
```bash
admin@raspberrypi:~ $ sudo nano /boot/config.txt
# Use the /dev/ttyAMA0 UART for user applications (GPS), not Bluetooth
dtoverlay=pi3-disable-bt
```

Those wishing to actually use Bluetooth with the software UART can do so, though with reduced efficiency, with:
```bash
# Use software UART for Bluetooth
enable_uart=1
dtoverlay=pi3-miniuart-bt
```

All changes to `/boot/config.txt` require a reboot to make it so. My configs do not use Bluetooth at all.
Though not strictly necessary, we can also disable the hciuart service that nominally attempts to talk to the UART; this may prevent some warnings in the logfiles:

```bash
admin@raspberrypi:~ $ sudo systemctl disable hciuart
Removed /etc/systemd/system/multi-user.target.wants/hciuart.service.
admin@raspberrypi:~ $ 
```

If this has been done correctly (and after a reboot - `admin@raspberrypi:~ $ sudo reboot`), we expect to see NMEA data on that serial port, which you can see directly with:
```bash
admi@raspberrypi:~ $ cat /dev/ttyAMA0
$GPGGA,052731.000,3343.3943,N,11749.3064,W,2,04,3.93,24.8,M,-34.2,M,0000,0000*65

$GPGSA,A,3,13,17,28,19,,,,,,,,,4.05,3.93,0.99*0C

$GPRMC,052731.000,A,3343.3943,N,11749.3064,W,0.84,313.46,240117,,,D*74
(...)
```

If you see the above, your serial port is configured correctly and it's seeing GPS data.
Do not move to the next step until you can see serial data.

## PPS (Pulse Per Second)
Linux has special kernel support for Pulse Per Second input via a GPIO pin to help synchronize the time: it associates a hyper-precise kernel timestamp with the rising edge of the PPS signal and makes it available to the application; it can then get highly accurate timestamps.

### Install pps-tools
These are a standard package, though not installed by default; they are added with:
```bash
admin@raspberrypi:~ $ sudo apt-get install pps-tools
```

### Enable PPS support in the kernel
We have to tell the Linux kernel that PPS support is desired, and which GPIO pin to use for it. As with the UART configuration, this is done in `/boot/config.txt` by adding this to the bottom:
```bash
admin@raspberrypi:~ $ sudo nano /boot/config.txt 
# enable GPS PPS
dtoverlay=pps-gpio,gpiopin=4
```

NOTE: different GPS hats use different GPIO pins for PPS; check your documentation to see whether it's using GPIO #4 or GPIO #18 and edit above as needed.

Do a `sudo reboot` after this editing.

### Test the PPS support
With this done, the special device `/dev/pps0` is available to poll the PPS signal, and it can be checked out with the `ppstest` program:
```bash
admin@raspberrypi:~ $ sudo ppstest /dev/pps0
trying PPS source "/dev/pps0"
found PPS source "/dev/pps0"
ok, found 1 source(s), now start fetching data...
time_pps_fetch() error -1 (Connection timed out)
time_pps_fetch() error -1 (Connection timed out)
```
`ppstest` timed out because my antenna did not have any GPS coverage. This was the intended result:
```bash
admin@raspberrypi:~ $ sudo ppstest /dev/pps0
trying PPS source "/dev/pps0"
found PPS source "/dev/pps0"
ok, found 1 source(s), now start fetching data...
source 0 - assert 1485236391.009463473, sequence: 148 - clear  0.000000000, sequence: 0
source 0 - assert 1485236392.009462426, sequence: 149 - clear  0.000000000, sequence: 0
source 0 - assert 1485236393.009459816, sequence: 150 - clear  0.000000000, sequence: 0
source 0 - assert 1485236394.009457259, sequence: 151 - clear  0.000000000, sequence: 0
source 0 - assert 1485236395.009456680, sequence: 152 - clear  0.000000000, sequence: 0
(...)
```

This should show one line every second; if it does, your Ultimate GPS hat is delivering PPS to the Linux kernel properly, and you have all the GPS inputs required: now we have to use them.

## Install and configure gpsd
NMEA is an acronym for the National Marine Electronics Association. NMEA existed well before GPS was invented. According to the NMEA website, the association was formed in 1957 by a group of electronic dealers to create better communications with manufacturers. Today in the world of GPS, NMEA is a standard data format supported by all GPS manufacturers, much like ASCII is the standard for digital computer characters in the computer world.

The purpose of NMEA is to give equipment users the ability to mix and match hardware and software. NMEA-formatted GPS data also makes life easier for software developers to write software for a wide variety of GPS receivers instead of having to write a custom interface for each GPS receiver. For example, VisualGPS software (free), accepts NMEA-formatted data from any GPS receiver and graphically displays it. Without a standard such as NMEA, it would be time-consuming and expensive to write and maintain such software.

There are a number of services that can decode and interpret the NMEA data coming from your Ultimate GPS hat, but the most popular by a fair measure is the open source `gpsd`. 
`gpsd` is a general-purpose daemon designed to interact with most types of GPS models using a wide variety of protocols. It is also capable of processing the PPS signals and sending timing information to `ntpd` via dedicated shared memory devices. One for pushing to `ntpd` absolute timestamp parsed from the NMEA messages (or supported equivalent), another for PPS timing data.

For more information about NMEA data, visit [https://www.gpsworld.com/what-exactly-is-gps-nmea-data/](https://www.gpsworld.com/what-exactly-is-gps-nmea-data/).

```bash
admin@raspberrypi:~ $ sudo apt-get install gpsd
```

### Create /etc/default/gpsd
This file contains the default parameters for the service when it starts, and this includes the required device names for the serial port and PPS device names. 

`gpsd` will definitely not work right without this.
```bash
admin@raspberrypi:~ $ sudo nano /etc/default/gpsd 
# /etc/default/gpsd
#
# Default settings for the gpsd init script and the hotplug wrapper.

# Start the gpsd daemon automatically at boot time
START_DAEMON="true"

# Use USB hotplugging to add new USB devices automatically to the daemon
USBAUTO="false"

# Devices gpsd should collect to at boot time.
# They need to be read/writeable, either by user gpsd or the group dialout.
DEVICES="/dev/ttyAMA0 /dev/pps0"

# Other options you want to pass to gpsd
#
# -n    don't wait for client to connect; poll GPS immediately

GPSD_OPTIONS="-n"
```

Key valus here are the `DEVICES=`, which provides the name of the serial port feeding NMEA data (`/dev/ttyAMA0`) and the pulse-per-second (`/dev/pps0`); they must appear in that order.
Also important is the `-n` value in `GPSD_OPTIONS=` this tells GPSD to start talking to the GPS device on startup and not to wait for the first client to attach.
Since we have a fulltime connection via the hardware serial port, there's no need not to start talking with the GPS unit immediately.

### Configure gpsd service
```bash
admin@raspberrypi:~ $ sudo systemctl enable gpsd
Synchronizing state of gpsd.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable gpsd
Created symlink /etc/systemd/system/multi-user.target.wants/gpsd.service → /lib/systemd/system/gpsd.service.
admin@raspberrypi:~ $ sudo systemctl start gpsd
admin@raspberrypi:~ $ 
```

### Check status
Assuming the "start" operation succeeded, check the status more generally via `systemctl`:
```
admin@raspberrypi:~ $ sudo systemctl status gpsd
● gpsd.service - GPS (Global Positioning System) Daemon
   Loaded: loaded (/lib/systemd/system/gpsd.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2020-01-09 17:47:55 GMT; 8min ago
 Main PID: 680 (gpsd)
    Tasks: 1 (limit: 2200)
   Memory: 452.0K
   CGroup: /system.slice/gpsd.service
           └─680 /usr/sbin/gpsd

Jan 09 17:47:55 raspberrypi systemd[1]: Starting GPS (Global Positioning System) Daemon...
Jan 09 17:47:55 raspberrypi systemd[1]: Started GPS (Global Positioning System) Daemon.
admin@raspberrypi:~ $ 
```
Here we're looking for the status is enabled and active and running, possibly with a few snippets of logfiles after.

At this point, if all is well, the `gpsd` service will be running and syncing with the GPS device. That doesn't necessarily mean that the GPS module has a fix on the satellites; that you can tell by looking at the Ultimate GPS hat for a brief red LED flash every 15 seconds. A LED that's flashing one second on/one second off has not yet acquired a fix.

Now for the moment of truth: we're going to install the `gpsmon` program to see what the daemon is doing. The `gpsmon` command connects to the `gpsd` server and displays output about the connected GPS.  You should see a list of satellites on the left (0 through 11) along with S/N ratios (some should be in the 30s or higher if you have good antenna placement).  You should see values for “Latitude” and “Longitude”.  If you have a good antenna positioning and you’re in the USA, you should see “FAA: D” in the middle box.  

The middle bottom box should also indicate a PPS value (a decimal number, which should be very small if your time on your machine is close to correct).
```bash
admin@raspberrypi:~ $ sudo apt-get install gpsd-clients
admin@raspberrypi:~ $ gpsmon

/dev/ttyAMA0                  NMEA0183>
┌──────────────────────────────────────────────────────────────────────────────┐
│Time: 2020-01-09T18:04:24.000Z Lat:  99 55' 88.01360" Non:  10 47' 66.07979" E│
└───────────────────────────────── Cooked TPV ─────────────────────────────────┘
┌──────────────────────────────────────────────────────────────────────────────┐
│ GPGGA GPGSA GPRMC GPZDA                                                      │
└───────────────────────────────── Sentences ──────────────────────────────────┘
┌──────────────────┐┌────────────────────────────┐┌────────────────────────────┐
│Ch PRN  Az El S/N ││Time:      828282.000       ││Time:      828288.000       │
│ 0                ││Latitude:     9999.2356 N   ││Latitude:  9999.2355        │
│ 1                ││Longitude:   66666.4633 E   ││Longitude: 66666.4629       │
│ 2                ││Speed:     0.56             ││Altitude:  73.7             │
│ 3                ││Course:    246.89           ││Quality:   1   Sats: 04     │
│ 4                ││Status:    A       FAA: D   ││HDOP:      3.55             │
│ 5                ││MagVar:                     ││Geoid:     40.6             │
│ 6                │└─────────── RMC ────────────┘└─────────── GGA ────────────┘
│ 7                │┌────────────────────────────┐┌────────────────────────────┐
│ 8                ││Mode: A3 Sats:              ││UTC:           RMS:         │
│ 9                ││DOP: H=3.55  V=0.93  P=3.67 ││MAJ:           MIN:         │
│10                ││TOFF:  0.388980357          ││ORI:           LAT:         │
│11                ││PPS: -0.000152647           ││LON:           ALT:         │
└────── GSV ───────┘└──────── GSA + PPS ─────────┘└─────────── GST ────────────┘

```
PS: Use CTRL + S to freeze the picture.

The top line summarizes the best information collected from the GPS receiver, including the time (in UTC) and the current location, and the bottom portion of the screen shows the more or less raw data from the GPS unit.
Of particular import are the lines ---- PPS offset --- because they indicate proper handling of the pulse-per-second input.

If the top of the `gpsmon` display shows:
```bash
+------------------------------------------------------------------------------+
|Time: 1980-01-08T14:14:50.092Z Lat: n/a               Lon: n/a                |
+--------------------------------- Cooked TPV ---------------------------------+
```

The lack of Lat/Lon indicates the GPS receiver has not obtained a satellite fix, and the time in `1980` (or `n/a`) indicates that it doesn't see one satellite to get a rough time.
Always check for the fix LED on the GPS hat itself, which should flash once every 15 seconds; if instead it's one second on/one second off, then it doesn't have a fix and there's not much that `gpsmon` can do for you.

Do not proceed to the next section until you get valid PPS input and valid GPS data showing up via gpsmon.

## Build and configure NTPsec
The stock `ntpd` shipped with your distribution is intended to be used as a client instance, not a server. It doesn't do 1PPS, and thereforce can't be used for precision timekeeping. Thus, we're going to build a better version from source. That version is NTPsec, which runs lighter and more securely and can do more accurate time stepping.

To read more about NTPSec vs NTP, go to https://www.ntpsec.org

To ensure the correct functioning of NTPsec, we should also disable the local time synchronization service. 
```bash
administrator@raspberrypi:~ $ sudo systemctl stop systemd-timesyncd.service
[sudo] password for administrator: 
Warning: The unit file, source configuration file or drop-ins of systemd-timesyncd.service changed on disk. Run 'systemctl daemon-reload' to reload units.
administrator@raspberrypi:~ $ sudo systemctl disable systemd-timesyncd.service
Removed /etc/systemd/system/sysinit.target.wants/systemd-timesyncd.service.
Removed /etc/systemd/system/dbus-org.freedesktop.timesync1.service.
```



Install the build prerequisites:

```bash
admin@raspberrypi:~ $ sudo apt install bison libcap-dev libssl-dev libreadline-dev git python-dev bc
```

Build NTPsec. The `--refclock` option says to include only shared-memory refclock support, excluding all other drivers (GPS, PTP, SHM, etc) `--refclock=all`:

```bash
admin@raspberrypi:~ $ cd /home/pi/
admin@raspberrypi:~ $ git clone https://gitlab.com/NTPsec/ntpsec.git
admin@raspberrypi:~ $ cd ntpsec/
admin@raspberrypi:~/ntpsec $ ./waf configure --refclock=shm
(...)
'configure' finished successfully (46.675s)
```
Finally, we build the source code:
```
admin@raspberrypi:~/ntpsec $ ./waf build
```
Install the binaries:
```bash
admin@raspberrypi:~/ntpsec $ sudo ./waf install
```

Delete the temporary working directory (NTPsec source code):
```bash
administrator@raspberrypi:~/ntpsec $ cd ..
administrator@raspberrypi:~ $ rm -r -f ntpsec/
```

Let’s check the NTPsec version number:
```bash
administrator@raspberrypi:~ $ sudo /usr/local/sbin/ntpd -V
ntpd ntpsec-1.1.8+ 2020-03-06T23:05:21Z (git rev f0bd02b25)
```
### Setting up the NTP Service (Systemd)
Perfect. Next, we set up NTPsec as a system service. For this, we create an `ntp` user and an `ntp` group with restricted rights. We also create folders for log files and certificates.

Because Raspbian no longer includes `ntpd`, we need to add a user for our NTPSec `ntpd` to run as, and create a couple of directories `ntpd` needs in place before running.

Create ntp directories:
```bash
administrator@raspberrypi:~ $ sudo mkdir -p /var/lib/ntp/certs
administrator@raspberrypi:~ $ sudo mkdir -p /var/log/ntpstats
```
Add system user `ntp`:
```bash
administrator@raspberrypi:~ $ sudo adduser --system --no-create-home --disabled-login --gecos '' ntp
```
Add group `ntp`:
```bash
administrator@raspberrypi:~ $ sudo addgroup --system ntp
```
Add user `ntp` to group `ntp`:
```bash
administrator@raspberrypi:~ $ sudo addgroup ntp ntp
```
Set folder permissions (recursive):
```bash
administrator@raspberrypi:~ $ sudo chown -R ntp:ntp /var/lib/ntp /var/log/ntpstats
```
Enable the NTPsec service:
```bash
administrator@raspberrypi:~ $ sudo systemctl enable ntpd
Created symlink /etc/systemd/system/multi-user.target.wants/ntpd.service → /lib/systemd/system/ntpd.service.
```
Now NTPsec is executed automatically when rebooting the system. At the moment the service is not running yet, because we have to configure it first.

### Feeding NTPD
The final part of this project configures the NTP (Network Time Protocol) daemon to consume the GPS time, possibly correlate it with other sources, and make accurate time available to the local machine and the rest of the network.

How does NTP get the precise time from `gpsd`? The answer is via shared memory. NTPD supports multiple kinds of time drivers, and one of them is a set of shared memory segments that feeds accurate time to all callers.
Note that `gpsd` also provides a TCP-based network service for all of the data it collects (certainly location data), but the network service is not useful for accurate time; the shared memory segment is far more precise for process-to-process communications.
The configuration is a little tricky and is not at all intuititive, so we'll do checking in steps.
First is the tool `ntpshmmon`, which monitors shared memory as if it were a client, and reports the counters found:
```bash
admin@raspberrypi:~ $ ntpshmmon
ntpshmmon version 1
#      Name Seen@                Clock                Real                 L Prec
sample NTP2 1579299789.000814106 1579299789.000017440 1579299789.000000000 0 -30
sample NTP2 1579299790.000537601 1579299790.000017393 1579299790.000000000 0 -30
sample NTP2 1579299791.000212137 1579299791.000017137 1579299791.000000000 0 -30
sample NTP2 1579299792.000893131 1579299792.000016570 1579299792.000000000 0 -30
(...)
```
The key values here are `NTP0` (which represents the time, but not displayed here), and `NTP2`, which represents the PPS; I believe the "Prec" (precision) column is what identifies this but am not sure. These NTP# values are key for the next step.

Create your own NTP configuration with this content as `ntp.conf`:

```bash
admin@raspberrypi:~/ntpsec $ sudo touch /etc/ntp.conf
admin@raspberrypi:~/ntpsec $ sudo nano /etc/ntp.conf

# /etc/ntp.conf, configuration for ntpd; see ntp.conf(5) for help


# You do need to talk to an NTP server or two (or three).
# server ntp.your-provider.example

# pool.ntp.org maps to about 1000 low-stratum NTP servers.  Your server will
# pick a different set every time it starts up.  Please consider joining the
# pool: <http://www.pool.ntp.org/join.html>
#pool 0.debian.pool.ntp.org iburst
#pool 1.debian.pool.ntp.org iburst
#pool 2.debian.pool.ntp.org iburst
#pool 3.debian.pool.ntp.org iburst

# Example 1: unsecured NTP
#server ntp1.glypnod.com iburst minpoll 3 maxpoll 6
 
# Example 2: NTS-secured NTP (default NTS-KE port (123); using certificate pool of the operating system)
server ntp1.glypnod.com iburst minpoll 3 maxpoll 6 nts
 
# Example 3: NTS-secured NTP (custom certificate and NTS-KE port)
#server nts3-e.ostfalia.de:443 iburst minpoll 3 maxpoll 6 nts ca /var/lib/ntp/certs/rootCaBundle.pem
 
# Example 4: NTS-secured NTP (skip DNS check)
#server nts3-e.ostfalia.de:443 iburst minpoll 3 maxpoll 6 nts ca /var/lib/ntp/certs/rootCaBundle.pem noval

# GPS PPS reference
server 127.127.28.2 prefer
fudge  127.127.28.2 refid PPS

# get time from SHM from gpsd; this seems working
server 127.127.28.0
fudge  127.127.28.0 refid GPS

 
# optional: allows a fast frequency error correction on startup
driftfile /var/lib/ntp/ntp.drift
 
# optional: 
# Enable this if you want statistics to be logged.
# statsdir /var/log/ntpstats/
statsdir /var/log/ntpstats
statistics loopstats peerstats clockstats rawstats
filegen loopstats file loopstats type day enable
filegen peerstats file peerstats type day enable
filegen clockstats file clockstats type day enable
filegen rawstats file rawstats type day enable
 
# optional: logging
logfile /var/log/ntp.log
logconfig =syncall +clockall +peerall +sysall

# Leap seconds definition provided by tzdata
leapfile /usr/share/zoneinfo/leap-seconds.list


# Access control configuration; see /usr/share/doc/ntp-doc/html/accopt.html for
# details.  The web page <http://support.ntp.org/bin/view/Support/AccessRestrictions>
# might also be helpful.
#
# Note that "restrict" applies to both servers and clients, so a configuration
# that might be intended to block requests from certain clients could also end
# up blocking replies from your own upstream servers.

# By default, exchange time with everybody, but don't allow configuration.
restrict -4 default kod notrap nomodify nopeer noquery limited
restrict -6 default kod notrap nomodify nopeer noquery limited

# Local users may interrogate the ntp server more closely.
restrict 127.0.0.1
restrict ::1

# Needed for adding pool entries
restrict source notrap nomodify noquery

# Clients from this (example!) subnet have unlimited access, but only if
# cryptographically authenticated.
#restrict 192.168.123.0 mask 255.255.255.0 notrust


# If you want to provide time to your local subnet, change the next line.
# (Again, the address is an example only.)
#broadcast 192.168.123.255

# If you want to listen to time broadcasts on your local subnet, de-comment the
# next lines.  Please do this only if you trust everybody on the network!
#disable auth
#broadcastclient
admin@raspberrypi:~/ntpsec $ 
```

In the first example, the Raspberry Pi synchronizes its clock with a public NTPsec time server using a classical unsecured NTP connection.

In the second example, the client communicates with the same time server via an NTS-secured NTP connection. The initial channel to the NTS-KE server uses the default port 123 TCP (currently implementation-specific). Since the NTPsec time server uses certificates issued by Let’s Encrypt, we do not need to set any additional parameters. To check the certificates, the client uses the local root CA pool (`/etc/ssl/certs/ca-certificates.crt`), which also allows the verification of certificates issued by Let’s Encrypt.

In the third example, we connect to the time server of the Ostfalia University, which is also NTS-capable. This uses TCP port 443 for the NTS-KE connection and uses self-signed test certificates. To check the server certificate we have to specify the root CA manually. The corresponding certificate can be downloaded by the following command:
```bash
sudo wget http://nts3-e.ostfalia.de/homePi/CLIENT/rootCaBundle.pem -P /var/lib/ntp/certs/
```

Caution: The time server of the Ostfalia University also publishes its private key, as it is a public test server. This time server should not be used for clock synchronization of productive systems.

The fourth example differs from the third one only from the deactivated domain validation. This can be useful when we running a local NTS server with certificates without a registered domain.

The other entries in the configuration file are optional and are used to record statistics and log files. The descriptions as well as the complete parameter list (incl. NTS) can be found in `ntp_conf.adoc`. Here only very briefly described:

```bash
server <name>:  The destination NTP time server (DNS name or IP address)
iburst:         Sends 8 NTP requests directly after startup
minpoll <val>:  Minimal request interval (power of two)
maxpoll <val>:  Maximal request interval (e.g: '6' means 2^6 = 64 sec)
nts:            [NTS] Enables NTS support
ca <file>:      [NTS] The trusted root CA certificate for the server
noval:          [NTS] Skips the DNS verification
```

Even though GPS provides highly accurate time, it's a best practice to have multiple time sources and let `ntpd` sort it out: if the GPS antenna gets disloged or blocked or something, we want at least some semblance of accurate time rather than just freewheeling with no real time source. 


We've also added in some fake server lines that hook into the shared memory portion. This is done with "servers" using IP addresses in the 127.127.t.u range, where "t" is the clock driver type and "x" is the unit number within that type. The type numbers are hardcoded in the program with several dozen driver numbers assigned.

A small sample of drivers:
* 127.127.4.u; Spectracom receiver-based clock
* 127.127.19.u; WWV/WWVH-based radio receiver
* 127.127.20.u; NMEA-based GPS clock
* 127.127.28.u; SHM driver
* 127.127.29.u; Trimble Palisade GPS

We only care about 127.127.28.u, which is the Shared Memory driver supported by `gpsd`, but these are not real IP addresses; don't bother trying to ping them.

The key values here are the final octets: the number represents the NTP# tag in shared memory, so 127.127.28.0 refers to NTP0 (the time source) and 127.127.28.2 represents NTP2 (the PPS signal). 

The refid tags are actually free text (1-4 chars) and can be anything you like; these show up in the "peers" display so you can tell where the time source got its data from. If you don't include a refid, it will show as .SHM., the tag for the driver.

* A reliable source has suggested that putting the PPS server+fudge stanzas first, followed by GPS, and then only preferring PPS, will yield a more reliable clock.


### Disable NTP support in DHCP
DHCP on many networks delivers time server config to their clients, allowing them to synchronize time properly along with the rest of the network. 
This is a good thing except for NTP time servers themselves.

Remove `ntp-servers` from `/etc/dhcp/dhclient.conf`:

```bash
admin@raspberrypi:~ $ sudo nano /etc/dhcp/dhclient.conf 
(...)
send host-name = gethostname();
request subnet-mask, broadcast-address, time-offset, routers,
        domain-name, domain-name-servers, domain-search, host-name,
        dhcp6.name-servers, dhcp6.domain-search, dhcp6.fqdn, dhcp6.sntp-servers,
        netbios-name-servers, netbios-scope, interface-mtu,
        rfc3442-classless-static-routes;
```

Delete these related files: `/lib/dhcpcd/dhcpcd-hooks/50-ntp.conf`
```bash
admin@raspberrypi:~ $ sudo mv /lib/dhcpcd/dhcpcd-hooks/50-ntp.conf /lib/dhcpcd/dhcpcd-hooks/50-ntp.bak
```
It probably requires reboot for this to take effect. At this point, you will be able to configure NTP support entirely without interference from the rest of the system.
```bash
admin@raspberrypi:~ $ sudo reboot
```
### Status of ntp
After the reboot, check the `ntp` status with `ntpq -p`:
```bash
admin@raspberrypi:~ $ ntpq -p
     remote                                   refid      st t when poll reach   delay   offset   jitter
=======================================================================================================
+ntp1.glypnod.com                        204.123.2.72     2 7   66   64    2 180.8374  -1.0179   0.0000                                                                     
*SHM(2)                                  .PPS.            0 l   56   64    1   0.0000  -0.0181   0.0000
xSHM(0)                                  .GPS.            0 l   15   64    3   0.0000 -506.770 300.2489
admin@raspberrypi:~ $ 
```
* `*` means that this is the preferred time server

### ntpstat
But how good is our time, compared to other sources? Install `ntpstat` to get an idea; `sudo apt-get install ntpstat`:

Here is another raspberry client using a server from a default pool. 
```bash
pi@halberry:~$ ntpstat
synchronised to NTP server (192.36.143.130) at stratum 2 
   time correct to within 196 ms
   polling server every 64 s
```

Compared to our stratum 1 server;
```bash
admin@raspberrypi:~ $ ntpstat
synchronised to UHF radio at stratum 1 
   time correct to within 2 ms
   polling server every 64 s
admin@raspberrypi:~ $ 
```

## Share our time
Now that this raspberry is syncing with a UFH radio at stratum 1, we want to make our raspberry available as a NTP server. This is already configured in our `ntp.conf` file, the section with access control configuration. You can check if our Raspberry is allowing NTP traffic with `netstat`:

```bash
administrator@raspberrypi:~ $ netstat -a | grep ntp                      
udp        0      0 localhost:ntp           0.0.0.0:*                          
udp        0      0 0.0.0.0:ntp             0.0.0.0:*                          
udp6       0      0 fe80::f347:77da:42d:ntp [::]:*                             
udp6       0      0 localhost:ntp           [::]:*                             
udp6       0      0 [::]:ntp                [::]:*    
```

### Configure PfSense
Interfaces > Assignments > VLANs > Click Add.

* Parent interface: Your LAN interface
* VLAN Tag: 28
* VLAN Priority: 0
* Description: NTP server
Click Save

Go Interfaces > Assignments > and select Interface Assignments.

Under Available network ports: select  `VLAN 28 on em0 - lan (NTP Server)` and click Add. 

Click on the newly added interface.

* Enable: [v] Enable interface
* Description: VLAN28_NTP
* IPv4 Configuration Type: Static IPv4
* IPv6 Configuration Type: None
* IPv4 Address: 192.168.28.1/30

Save and Apply Changes

Go to Services > DHCP Server and select your interface VLAN28_NTP.

* Enable DHCP server on VLAN28_NTP interface
* Range: From 192.168.28.2 To: 192.168.28.2

Go to Services > NTP and Add our NTP server with an address of 192.168.28.2 and select Prefer. 

You should now see under Status > NTP that our NTP server at `192.168.28.2` is our active Peer. 









# USB Dongle 
The following information is now regarded deprecated; how I tried to do it with a USB dongle and the regular ntp daemon, not the secure one.
## Install GPS packages
`gpsd` is a general-purpose daemon designed to interact with most types of GPS models using a wide variety of protocols. It is also capable of processing the PPS signals and sending timing information to ntpd via dedicated shared memory devices. One for pushing to ntpd absolute timestamp parsed from the NMEA messages (or supported equivalent), another for PPS timing data. 

ntpd will see them as two different SHM devices (Shared Host Memory).

```bash
admin@raspberrypi:~ $ sudo apt-get install gpsd gpsd-clients python-gps ntp pps-tools
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following additional packages will be installed:
  adwaita-icon-theme fontconfig fontconfig-config fonts-dejavu-core gtk-update-icon-cache hicolor-icon-theme libatk1.0-0 libatk1.0-data libavahi-client3 libblas3 libbluetooth3 libcairo2 libcroco3 libcups2 libdatrie1 libevent-core-2.1-6
  libevent-pthreads-2.1-6 libfontconfig1 libgail-common libgail18 libgdk-pixbuf2.0-0 libgdk-pixbuf2.0-bin libgdk-pixbuf2.0-common libgfortran5 libgps23 libgraphite2-3 libgtk2.0-0 libgtk2.0-bin libgtk2.0-common libharfbuzz0b libjbig0 liblapack3 libopts25
  libpango-1.0-0 libpangocairo-1.0-0 libpangoft2-1.0-0 libpixman-1-0 librsvg2-2 librsvg2-common libthai-data libthai0 libtiff5 libwebp6 libxcb-render0 libxcb-shm0 libxcomposite1 libxcursor1 libxdamage1 libxfixes3 libxi6 libxinerama1 libxrandr2 libxrender1
  python-cairo python-gobject-2 python-gtk2 python-numpy python-pkg-resources sntp
Suggested packages:
  cups-common gvfs librsvg2-bin ntp-doc python-gobject-2-dbg python-gtk2-doc gfortran python-dev python-pytest python-numpy-dbg python-numpy-doc python-setuptools
The following NEW packages will be installed:
  adwaita-icon-theme fontconfig fontconfig-config fonts-dejavu-core gpsd gpsd-clients gtk-update-icon-cache hicolor-icon-theme libatk1.0-0 libatk1.0-data libavahi-client3 libblas3 libbluetooth3 libcairo2 libcroco3 libcups2 libdatrie1 libevent-core-2.1-6
  libevent-pthreads-2.1-6 libfontconfig1 libgail-common libgail18 libgdk-pixbuf2.0-0 libgdk-pixbuf2.0-bin libgdk-pixbuf2.0-common libgfortran5 libgps23 libgraphite2-3 libgtk2.0-0 libgtk2.0-bin libgtk2.0-common libharfbuzz0b libjbig0 liblapack3 libopts25
  libpango-1.0-0 libpangocairo-1.0-0 libpangoft2-1.0-0 libpixman-1-0 librsvg2-2 librsvg2-common libthai-data libthai0 libtiff5 libwebp6 libxcb-render0 libxcb-shm0 libxcomposite1 libxcursor1 libxdamage1 libxfixes3 libxi6 libxinerama1 libxrandr2 libxrender1
  ntp pps-tools python-cairo python-gobject-2 python-gps python-gtk2 python-numpy python-pkg-resources sntp
0 upgraded, 64 newly installed, 0 to remove and 1 not upgraded.
Need to get 37.7 MB of archives.
After this operation, 110 MB of additional disk space will be used.
Do you want to continue? [Y/n] y

```
### Reboot
```bash
admin@raspberrypi:~ $ sudo reboot
```

## LSUSB
```bash
ubuntu@ubuntu:~$ lsusb
Bus 001 Device 003: ID 0424:ec00 Standard Microsystems Corp. SMSC9512/9514 Fast Ethernet Adapter
Bus 001 Device 002: ID 0424:9514 Standard Microsystems Corp. SMC9514 Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
ubuntu@ubuntu:~$ 
```

### Insert USB GPS dongle
```bash
ubuntu@ubuntu:~$ lsusb
Bus 001 Device 006: ID 067b:2303 Prolific Technology, Inc. PL2303 Serial Port
Bus 001 Device 003: ID 0424:ec00 Standard Microsystems Corp. SMSC9512/9514 Fast Ethernet Adapter
Bus 001 Device 002: ID 0424:9514 Standard Microsystems Corp. SMC9514 Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
ubuntu@ubuntu:~$ 
```

## Edit “/etc/default/gpsd”
```bash
admin@raspberrypi:~ $ sudo nano /etc/default/gpsd

# Default settings for the gpsd init script and the hotplug wrapper.

# Start the gpsd daemon automatically at boot time
START_DAEMON="true"

# Use USB hotplugging to add new USB devices automatically to the daemon
USBAUTO="true"

# Devices gpsd should collect to at boot time.
# They need to be read/writeable, either by user gpsd or the group dialout.
DEVICES="/dev/ttyUSB0"

# Other options you want to pass to gpsd
GPSD_OPTIONS="-F /var/run/gpsd.sock -b -n"

#GPSD_SOCKET="/var/run/gpsd.sock"
```

## Restart the gpsd Service     
```bash
admin@raspberrypi:~ $ sudo systemctl enable gpsd
Synchronizing state of gpsd.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable gpsd
Created symlink /etc/systemd/system/multi-user.target.wants/gpsd.service → /lib/systemd/system/gpsd.service.
admin@raspberrypi:~ $ sudo systemctl start gpsd
admin@raspberrypi:~ $ sudo systemctl status gpsd
● gpsd.service - GPS (Global Positioning System) Daemon
   Loaded: loaded (/lib/systemd/system/gpsd.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2020-01-05 21:18:31 GMT; 4s ago
  Process: 694 ExecStart=/usr/sbin/gpsd $GPSD_OPTIONS $DEVICES (code=exited, status=0/SUCCESS)
 Main PID: 695 (gpsd)
    Tasks: 2 (limit: 2200)
   Memory: 1.5M
   CGroup: /system.slice/gpsd.service
           └─695 /usr/sbin/gpsd -F /var/run/gpsd.sock -b -n /dev/ttyUSB0

Jan 05 21:18:31 raspberrypi systemd[1]: Starting GPS (Global Positioning System) Daemon...
Jan 05 21:18:31 raspberrypi systemd[1]: Started GPS (Global Positioning System) Daemon.
admin@raspberrypi:~ $ 
```

### cgps -s
At this point, you should be able to see a text-mode output from your GPS receiver by running the command `cgps -s`. 
This will display the details of your current location with values, something like the following:
```bash
admin@raspberrypi:~ $ cgps -s

┌───────────────────────────────────────────┐┌─────────────────────────────────┐
│    Time:       2020-01-05T16:11:12.000Z   ││PRN:   Elev:  Azim:  SNR:  Used: │
│    Latitude:    00.00000000 N             ││  33    12    009    23      Y   │
│    Longitude:   00.00000000 E             ││  21    41    066    38      Y   │
│    Altitude:   900.100 m                  ││  55    46    098    19      Y   │
│    Speed:      0.96 kph                   ││  25    07    237    31      Y   │
│    Heading:    180.0 deg (true)           ││  28    13    061    35      Y   │
│    Climb:      6.00 m/min                 ││  10    18    281    00      N   │
│    Status:     3D FIX (4 secs)            ││  12    45    224    00      N   │
│    Longitude Err:   +/- 19 m              ││  13    03    159    00      N   │
│    Latitude Err:    +/- 18 m              ││  15    21    184    00      N   │
│    Altitude Err:    +/- 62 m              ││  22    03    255    00      N   │
│    Course Err:      n/a                   ││  24    77    217    00      N   │
│    Speed Err:       +/- 139 kph           ││  32    21    315    00      N   │
│    Time offset:     -4344444.984          ││                                 │
│    Grid Square:     Q071pq                ││                                 │
└───────────────────────────────────────────┘└─────────────────────────────────┘
```

## Telling NTP the seconds from the GPS
Now that gpsd is working, we can edit the NTP configuration to add a type 28 reference clock which will make NTP look at the shared memory created by `gpsd`. 

While we are at it, we comment out all our `pool` lines as well, because we are not using internet for time sync anymore.

Edit `/etc/ntp.conf`:
```bash
admin@raspberrypi:~ $ sudo nano /etc/ntp.conf

(...)

# You do need to talk to an NTP server or two (or three).
#server ntp.your-provider.example

# pool.ntp.org maps to about 1000 low-stratum NTP servers.  Your server will
# pick a different set every time it starts up.  Please consider joining the
# pool: <http://www.pool.ntp.org/join.html>
pool 0.debian.pool.ntp.org iburst
pool 1.debian.pool.ntp.org iburst
pool 2.debian.pool.ntp.org iburst
pool 3.debian.pool.ntp.org iburst

# GPS NTP
server 127.127.28.0 minpoll 4
fudge 127.127.28.0  time1 0.183 refid NMEA
server 127.127.28.1 minpoll 4 prefer
fudge 127.127.28.1  time1 0.183 refid PPS

# Use Ubuntu's ntp server as a fallback.
# pool ntp.ubuntu.com

```
## Restart the ntp service
```bash
admin@raspberrypi:~ $ sudo systemctl enable ntp
Synchronizing state of ntp.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable ntp
admin@raspberrypi:~ $ sudo systemctl start ntp
admin@raspberrypi:~ $ sudo systemctl status ntp
● ntp.service - Network Time Service
   Loaded: loaded (/lib/systemd/system/ntp.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2020-01-05 21:14:58 GMT; 8min ago
     Docs: man:ntpd(8)
 Main PID: 514 (ntpd)
    Tasks: 2 (limit: 2200)
   Memory: 1.9M
   CGroup: /system.slice/ntp.service
           └─514 /usr/sbin/ntpd -p /var/run/ntpd.pid -g -u 109:114

Jan 05 21:15:02 raspberrypi ntpd[514]: Soliciting pool server 37.44.185.42
Jan 05 21:15:03 raspberrypi ntpd[514]: Soliciting pool server 91.189.182.32
Jan 05 21:15:03 raspberrypi ntpd[514]: Soliciting pool server 162.159.200.123
Jan 05 21:15:03 raspberrypi ntpd[514]: Soliciting pool server 162.159.200.1
Jan 05 21:15:04 raspberrypi ntpd[514]: Soliciting pool server 185.42.170.200
Jan 05 21:15:04 raspberrypi ntpd[514]: Soliciting pool server 2001:67c:24e4:33::36
Jan 05 21:15:17 raspberrypi ntpd[514]: receive: Unexpected origin timestamp 0xe1bcd05a.605a299f does not match aorg 0000000000.00000000 from server@193.150.22.36 xmt 0xe1bcd065.5cb17fde
Jan 05 21:15:17 raspberrypi ntpd[514]: receive: Unexpected origin timestamp 0xe1bcd05a.605492a6 does not match aorg 0000000000.00000000 from server@51.174.198.198 xmt 0xe1bcd065.5d08dbe7
Jan 05 21:15:17 raspberrypi ntpd[514]: receive: Unexpected origin timestamp 0xe1bcd05a.605763a9 does not match aorg 0000000000.00000000 from server@192.36.143.130 xmt 0xe1bcd065.5e6f150d
Jan 05 21:20:50 raspberrypi ntpd[514]: kernel reports TIME_ERROR: 0x41: Clock Unsynchronized
admin@raspberrypi:~ $ sudo reboot
```


## ntpq -p
After a bit (15 minutes or so), you should see `SHM(0)` selected, which will be indicated by an asterisk in front of that line.
It should display time from `SHM(0)` & `SHM(1)`.

```bash
admin@raspberrypi:~ $ ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 0.debian.pool.s .POOL.          16 p    -   64    0    0.000    0.000   0.002
 1.debian.pool.s .POOL.          16 p    -   64    0    0.000    0.000   0.002
 2.debian.pool.s .POOL.          16 p    -   64    0    0.000    0.000   0.002
 3.debian.pool.s .POOL.          16 p    -   64    0    0.000    0.000   0.002
#SHM(0)          .NMEA.           0 l   15   16   77    0.000  -194.69  19.948
 SHM(1)          .PPS.            0 l    -   16    0    0.000    0.000   0.000
+198.51-174-198. 195.106.111.68   2 u   36   64   17    3.068  308.754   0.368
-77-95-77-250.bb 194.58.202.20    2 u   34   64   17    5.036  308.356   0.469
+time100.stupi.s .PPS.            1 u   31   64   17   15.548  308.568   0.385
-ntp.coreless.ne 194.58.203.148   2 u   32   64   17    1.555  308.768   0.282
#87.248.15.31 (i 194.58.203.148   2 u   34   64   17    4.355  308.928   0.444
+162.159.200.123 10.83.8.95       3 u   37   64    7    1.250  308.311   0.299
+162.159.200.1   10.83.8.95       3 u   38   64    7    1.347  308.351   0.264
-ntp.ixo.no      77.40.226.114    2 u   37   64    7    1.980  308.689   0.226
#ntp-ext.cosng.n 146.213.3.180    2 u   31   64   17    4.414  308.696   0.088
*time.bjolab.net .GPS.            1 u   35   64   17   23.462  308.589   0.333
+ntp.fnutt.net   192.36.143.150   2 u   32   64   17    2.141  308.595   0.158
#ntp-ext.cosng.n 146.213.3.180    2 u   36   64   17    4.297  308.389   0.251
+nux.hackeriet.n 77.40.226.114    2 u   34   64   17    1.902  308.382   0.458
#ntp-ext.cosng.n 146.213.3.180    2 u   34   64   17    4.398  308.790   0.159
```

## Make it standalone
Comment out all our `pool` lines in `/etc/ntp.conf`, because we are not using internet for time sync anymore.

### Edit `/etc/ntp.conf`:
```bash
admin@raspberrypi:~ $ sudo nano /etc/ntp.conf
(...)
#pool 0.debian.pool.ntp.org iburst
#pool 1.debian.pool.ntp.org iburst
#pool 2.debian.pool.ntp.org iburst
#pool 3.debian.pool.ntp.org iburst
(...)
```
Save and then issue `sudo reboot` once more.

### What does ntpq -p say now?
```bash
admin@raspberrypi:~ $ ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*SHM(0)          .NMEA.           0 l   14   16    1    0.000  -50.303   0.002
 SHM(1)          .PPS.            0 l    -   16    0    0.000    0.000   0.000
```

# Fault finding


## lsmod
Is pps modules loaded?
```bash
admin@raspberrypi:~ $ lsmod | grep pps
pps_ldisc              16384  2
pps_core               20480  2 pps_ldisc
```

```bash
admin@raspberrypi:~ $ dmesg | grep pps
[    8.850684] pps_core: LinuxPPS API ver. 1 registered
[    8.850698] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    8.855854] pps_ldisc: PPS line discipline registered
[    8.857858] pps pps0: new PPS source usbserial0
[    8.857984] pps pps0: source "/dev/ttyUSB0" added
admin@raspberrypi:~ $ 
```

## Are you synchronized?
```
admin@raspberrypi:~ $ ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 SHM(1)          .PPS.            0 l    -   16    0    0.000    0.000   0.000
 SHM(0)          .NMEA.           0 l    -   16    0    0.000    0.000   0.000
admin@raspberrypi:~ $ ntpq -pn
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 127.127.28.1    .PPS.            0 l    -   16    0    0.000    0.000   0.000
 127.127.28.0    .NMEA.           0 l    -   16    0    0.000    0.000   0.000
```
If you do not have a `*` to the left of either source, you are not synchronized.

## Check the GPS status
Do your GPS have a 3D FIX?
```bash
admin@raspberrypi:~ $ cgps -s

┌───────────────────────────────────────────┐┌─────────────────────────────────┐
│    Time:       2020-01-05T22:14:54.110Z   ││PRN:   Elev:  Azim:  SNR:  Used: │
│    Latitude:   n/a                        ││   5    23    042    08      Y   │
│    Longitude:  n/a                        ││  21    61    169    12      Y   │
│    Altitude:   n/a                        ││  27    14    264    22      Y   │
│    Speed:      n/a                        ││  29    41    087    19      Y   │
│    Heading:    n/a                        ││  31    16    210    22      Y   │
│    Climb:      n/a                        ││ 193    11    040    12      Y   │
│    Status:     NO FIX (18 secs)           ││   9    07    321    00      N   │
│    Longitude Err:   n/a                   ││  16    50    288    00      N   │
│    Latitude Err:    n/a                   ││  20    07    156    00      N   │
│    Altitude Err:    n/a                   ││  25    02    138    04      N   │
│    Course Err:      n/a                   ││  26    67    240    00      N   │
│    Speed Err:       n/a                   ││                                 │
│    Time offset:     -366.660              ││                                 │
│    Grid Square:     n/a                   ││                                 │
└───────────────────────────────────────────┘└─────────────────────────────────┘
```
Try to move your GPS receiver. 

## gpsmon
The `gpsmon` command connects to the `gpsd` server and displays output about the connected GPS.  You should see a list of satellites on the left (0 through 11) along with S/N ratios (some should be in the 30s or higher if you have good antenna placement).  You should see values for “Latitude” and “Longitude”.  If you have a good antenna positioning and you’re in the USA, you should see “FAA: D” in the middle box.  The middle bottom box should also indicate a PPS value (a decimal number, which should be very small if your time on your machine is close to correct).
```bash
admin@raspberrypi:~ $ gpsmon
tcp://localhost:2947          SiRF>
(82) {"class":"VERSION","release":"3.17","rev":"3.17","proto_major":3,"proto_minor":12}
(198) {"class":"DEVICES","devices":[{"class":"DEVICE","path":"/dev/ttyUSB0","driver":"SiRF","activated":"2020-01-08T00:19:18.149Z","flags":1,"native":1,"bps":4800,"par
ity":"N","stopbits":1,"cycle":1.00}]}
(122) {"class":"WATCH","enable":true,"json":false,"nmea":false,"raw":2,"scaled":false,"timing":false,"split24":false,"pps":true}

┌─────────── X ────── Y ────── Z ────────── North ──── East ───── Alt ─────────┐
│Pos:  2819090   266717  9191276 m       44.92023° 135.78942°     59 m         │
│Vel:      0.0      0.0      0.0 m/s          0.0       0.0       0.0 climb m/s│
│Time: 2020-01-08T00:16:49.000Z Leap: ??     Heading:   0.0°      0.0 speed m/s│
│Fix:  6 = 16  8 26  7 15 13                          HDOP: 1.0  M1: 04  M2: 02│
└─────────────────────── Packet type 2 (0x02) ─────────────────────────────────┘
┌ Measured Tracker ──────────┐┌ Firmware Version ──────────────────────────────┐
│Ch PRN  Az El Stat  C/N ? SF││                                                │
│ 0   1 185 37 00ad 18.9     │└─────── Packet Type 6 (0x06) ───────────────────┘
│ 1   7  54 18 00ad 22.8     │┌ CPU Throughput ────────────────────────────────┐
│ 2  29  78 44 00ac  3.9     ││Max: 0.000  Lat: 0.000  Time: 0.000   MS:  0    │
│ 3   2 209 18 00bf 28.3 T   │└─────── Packet type 9 (0x09) ───────────────────┘
│ 4  20 362 72 00ad  9.1     │┌ Clock Status ──────────────────────────────────┐
│ 5  11  24 14 00ad 15.4     ││SVs:    Drift:        Bias:                     │
│ 6  19  95 43 00bf 27.8 T   ││GPS Time:             PPS:                      │
│ 7  11  54 39 008c  8.3     │└─────── Packet type 7 (0x07) ───────────────────┘
│ 8  23  81 19 00ad 21.6     │┌ Visible List ──────────────────────────────────┐
│ 9  21 212 53 0000  0.0     ││                                                │
│10  20 140 12 0000  0.0     │└─────── Packet type 13 (0x0D) ──────────────────┘
│11  80 308 18 0000  0.0     │┌ DGPS Status ───────────────────────────────────┐
└─── Packet Type 4 (0x04) ───┘│                                                │
                              └─────── Packet type 27 (0x1B) ──────────────────┘

```


## Check that things look right
```
admin@raspberrypi:~ $ watch ntpq -p
Every 2.0s: ntpq -p                                                                                                                                                                                                            raspberrypi: Sun Jan  5 21:50:50 2020

     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 SHM(1)          .PPS.            0 l    -   16    0    0.000    0.000   0.000
*SHM(0)          .NMEA.           0 l   50   16  270    0.000  -23.769   7.782

```
`*` is the source you are synchronized to (syspeer).


## ntpstat
```bash
admin@raspberrypi:~ $ sudo apt-get install ntpstat
admin@raspberrypi:~ $ ntpstat 
Unable to talk to NTP daemon. Is it running?
admin@raspberrypi:~ $ ntpstat
synchronised to UHF radio at stratum 1 
   time correct to within 8002 ms
   polling server every 16 s
admin@raspberrypi:~ $ 

```

## ppscheck
```
admin@raspberrypi:~ $ ppscheck /dev/ttyUSB0 
open(/dev/ttyUSB0) failed: 16 Device or resource busy
admin@raspberrypi:~ $ sudo systemctl stop gpsd
Warning: Stopping gpsd.service, but it can still be activated by:
  gpsd.socket
admin@raspberrypi:~ $ ppscheck /dev/ttyUSB0 
```

## ppstest
```bash
admin@raspberrypi:~ $ ls -l /dev/pps0 -l
crw------- 1 root root 240, 0 Jan  7 01:13 /dev/pps0
admin@raspberrypi:~ $ ppstest /dev/pps0
trying PPS source "/dev/pps0"
unable to open device "/dev/pps0" (Permission denied)
admin@raspberrypi:~ $ sudo ppstest /dev/pps0
trying PPS source "/dev/pps0"
found PPS source "/dev/pps0"
ok, found 1 source(s), now start fetching data...
time_pps_fetch() error -1 (Connection timed out)
time_pps_fetch() error -1 (Connection timed out)
time_pps_fetch() error -1 (Connection timed out)
^C
admin@raspberrypi:~ $ sudo systemctl stop gpsd
Warning: Stopping gpsd.service, but it can still be activated by:
  gpsd.socket
admin@raspberrypi:~ $ sudo ppstest /dev/pps0
trying PPS source "/dev/pps0"
unable to open device "/dev/pps0" (No such file or directory)
admin@raspberrypi:~ $ sudo ppstest /dev/pps0
trying PPS source "/dev/pps0"
unable to open device "/dev/pps0" (No such file or directory)
admin@raspberrypi:~ $ sudo systemctl start gpsd
admin@raspberrypi:~ $ sudo ppstest /dev/pps0
trying PPS source "/dev/pps0"
found PPS source "/dev/pps0"
ok, found 1 source(s), now start fetching data...
time_pps_fetch() error -1 (Connection timed out)
```
## Windows client settings
```
C:\Users\Don Pablo>w32tm /query /status
Leap Indicator: 0(no warning)
Stratum: 4 (secondary reference - syncd by (S)NTP)
Precision: -23 (119.209ns per tick)
Root Delay: 0.1828534s
Root Dispersion: 8.0194936s
ReferenceId: 0x28515E41 (source IP:  49.81.94.65)
Last Successful Sync Time: 07.03.2020 01:20:21
Source: time.windows.com,0x9
Poll Interval: 10 (1024s)


C:\Users\Don Pablo>
```
Start > Control Panel > Clock, Language, and Region.
Click the icon Date and Time. There, switch to the tab named Internet Time. To adjust available settings, you need to click the button "Change settings..." and add the ip to your Raspberry Pi for testing:

```
C:\Users\Don Pablo>w32tm /query /status
Leap Indicator: 0(no warning)
Stratum: 2 (secondary reference - syncd by (S)NTP)
Precision: -23 (119.209ns per tick)
Root Delay: 0.0007472s
Root Dispersion: 7.9968337s
ReferenceId: 0xC0A8056D (source IP:  192.168.53.234)
Last Successful Sync Time: 07.03.2020 01:24:11
Source: 192.168.53.234,0x9
Poll Interval: 10 (1024s)


C:\Users\Don Pablo>
```



## Authors
Mr. Johnson

## Acknowledgments
* [https://www.cyberciti.biz/faq/linux-unix-bsd-is-ntp-client-working/](https://www.cyberciti.biz/faq/linux-unix-bsd-is-ntp-client-working/)
* [http://www.linuxpps.org/pipermail/discussions/2008-July/002068.html](http://www.linuxpps.org/pipermail/discussions/2008-July/002068.html)
* [http://lkml.iu.edu/hypermail/linux/kernel/0902.0/03213.html](http://lkml.iu.edu/hypermail/linux/kernel/0902.0/03213.html)
* [https://www.lammertbies.nl/comm/info/gps-time](https://www.lammertbies.nl/comm/info/gps-time)
* [https://community.raspberryshake.org/t/is-my-gps-working/691/7](https://community.raspberryshake.org/t/is-my-gps-working/691/7)
* [https://weberblog.net/setting-up-nts-secured-ntp-with-ntpsec/](https://weberblog.net/setting-up-nts-secured-ntp-with-ntpsec/)
* [https://thepihut.com/blogs/raspberry-pi-tutorials/how-to-change-the-default-account-username-and-password](https://thepihut.com/blogs/raspberry-pi-tutorials/how-to-change-the-default-account-username-and-password)
* [https://www.howtovmlinux.com/articles/rasberry-pi/enable-root-login-and-change-password-raspberrypi.html](https://www.howtovmlinux.com/articles/rasberry-pi/enable-root-login-and-change-password-raspberrypi.html)
* [https://howchoo.com/g/odizmddly2j/change-the-password-on-your-raspberry-pi](https://howchoo.com/g/odizmddly2j/change-the-password-on-your-raspberry-pi)
* [https://www.techcoil.com/blog/how-to-setup-raspbian-buster-lite-for-raspberry-pi-server-projects/](https://www.techcoil.com/blog/how-to-setup-raspbian-buster-lite-for-raspberry-pi-server-projects/)
* [https://developers.redhat.com/blog/2017/02/22/how-to-build-a-stratum-1-ntp-server-using-a-raspberry-pi/](https://developers.redhat.com/blog/2017/02/22/how-to-build-a-stratum-1-ntp-server-using-a-raspberry-pi/)
* [https://coderich.net/2019/10/01/raspberry-pi-3-stratum-1-ntp-server-2/](https://coderich.net/2019/10/01/raspberry-pi-3-stratum-1-ntp-server-2/)
* [https://gpsd.gitlab.io/gpsd/gpsd-time-service-howto.html](https://gpsd.gitlab.io/gpsd/gpsd-time-service-howto.html)
* [https://www.satsignal.eu/ntp/Raspberry-Pi-NTP.html](https://www.satsignal.eu/ntp/Raspberry-Pi-NTP.html)
* [http://doc.ntp.org/4.2.8/drivers/driver28.html](http://doc.ntp.org/4.2.8/drivers/driver28.html)
* [https://unix.stackexchange.com/questions/351511/ntp-is-not-syncing-with-gps](https://unix.stackexchange.com/questions/351511/ntp-is-not-syncing-with-gps)
* [https://www.arednmesh.org/content/easy-gps-ntp-server-howto](https://www.arednmesh.org/content/easy-gps-ntp-server-howto)
* [https://www.raspberrypi.org/forums/viewtopic.php?t=191050](https://www.raspberrypi.org/forums/viewtopic.php?t=191050)
* [https://www.kernel.org/doc/html/latest/driver-api/pps.html](https://www.kernel.org/doc/html/latest/driver-api/pps.html)
* [https://gpsd.gitlab.io/gpsd/installation.html](https://gpsd.gitlab.io/gpsd/installation.html)
* [https://www.cyberciti.biz/faq/linux-unix-bsd-is-ntp-client-working/](https://www.cyberciti.biz/faq/linux-unix-bsd-is-ntp-client-working/)
* [http://www.linuxstall.com/how-to-display-a-digital-clock-in-linux-terminal/](http://www.linuxstall.com/how-to-display-a-digital-clock-in-linux-terminal/)
* [https://openwrt.org/docs/guide-user/services/ntp/gps](https://openwrt.org/docs/guide-user/services/ntp/gps)
* [https://forum.openwrt.org/t/lede-as-stratum-1-ntp-server-using-usb-gps/1997](https://forum.openwrt.org/t/lede-as-stratum-1-ntp-server-using-usb-gps/1997)
* [http://support.ntp.org/bin/view/Support/AccessRestrictions](http://support.ntp.org/bin/view/Support/AccessRestrictions)
* [https://digitalbarbedwire.com/2016/12/26/a-minimal-raspberry-pi-3-gps-time-server/](https://digitalbarbedwire.com/2016/12/26/a-minimal-raspberry-pi-3-gps-time-server/)
* [https://www.systutorials.com/docs/linux/man/1-gpsmon/](https://www.systutorials.com/docs/linux/man/1-gpsmon/)
* [https://www.roundsolutions.com/forum/index.php?thread/4077-pps-signal-without-fix/](https://www.roundsolutions.com/forum/index.php?thread/4077-pps-signal-without-fix/)
* [https://www.gpsworld.com/what-exactly-is-gps-nmea-data/](https://www.gpsworld.com/what-exactly-is-gps-nmea-data/)
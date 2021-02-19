---
layout: default
title: Grafana dashboard for pfSense
parent: pfSense
nav_order: 2
---

# Grafana dashboard for pfSense
{: .no_toc }
This is how I used Grafana to display dashboard for vitals from my PfSense firewall. I was able to do this because of this VictorRobellini's work: [https://github.com/VictorRobellini/pfSense-Dashboard](https://github.com/VictorRobellini/pfSense-Dashboard)

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
I am using an Ubuntu 20.04 installation on a virtual machine deployed from Proxmox. 

### Prerequsites
* Proxmox Virtual Environment 6.1-5
* Ubuntu 20.04 LTS
* PfSense 2.4.5-RELEASE-p1 (amd64) 
* InfluxDB 1.8.2
* Telegraf 0.9_4

---

## Allocating a VM in Proxmox
Log in to your Proxmox in a web-browser and create a new virtal machine. 2GiB RAM and 32GiB harddrive is enough.

Power on the Virtual machine and follow this guide for an excellent guide to installing ubuntu server: [https://tutorials.ubuntu.com/tutorial/tutorial-install-ubuntu-server-1604](https://tutorials.ubuntu.com/tutorial/tutorial-install-ubuntu-server-1604#0) 
(even though the link is for 1604, almost the same applies for 20.04).

Remember to select to install `OpenSSH server` under the installation of Ubuntu.

---

## Some things first ..
### update && upgrade
Update and upgrade:
```bash
torkel@gaard:~$ sudo apt-get update
torkel@gaard:~$ sudo apt-get upgrade
```

### Set your time zone
```bash
torkel@gaard:~$ date
Sat 11 Jan 21:22:53 GMT 2020
torkel@gaard:~$ sudo dpkg-reconfigure tzdata

Current default time zone: 'Europe/Paris'
Local time is now:      Sat Jan 11 22:24:07 CET 2020.
Universal Time is now:  Sat Jan 11 21:24:07 UTC 2020.

torkel@gaard:~$ date
Sat 11 Jan 22:24:16 CET 2020
```

### Disable IPv6
```bash
torkel@gaard:~$ sudo nano /etc/default/grub
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
torkel@gaard:~$ sudo update-grub
```

### Change NTP
Add your preferred NTP server:
```bash
torkel@gaard:~$  sudo nano /etc/systemd/timesyncd.conf 
[sudo] password for torkel:
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
torkel@gaard:~$ sudo systemctl restart systemd-timesyncd
```

Check NTP status:
```bash
torkel@gaard:~$ systemctl status systemd-timesyncd
● systemd-timesyncd.service - Network Time Synchronization
   Loaded: loaded (/lib/systemd/system/systemd-timesyncd.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2020-07-27 13:54:28 CEST; 50s ago
     Docs: man:systemd-timesyncd.service(8)
 Main PID: 626 (systemd-timesyn)
   Status: "Synchronized to time server 192.168.78.1:123 (192.168.78.1)."
    Tasks: 2 (limit: 2317)
   CGroup: /system.slice/systemd-timesyncd.service
           └─626 /lib/systemd/systemd-timesyncd

Jul 27 13:54:28 torkel systemd[1]: Starting Network Time Synchronization...
Jul 27 13:54:28 torkel systemd[1]: Started Network Time Synchronization.
Jul 27 13:54:32 torkel systemd-timesyncd[626]: No network connectivity, watching for changes.
Jul 27 13:54:59 torkel systemd-timesyncd[626]: Synchronized to time server 192.168.78.1:123 (192.168.78.1).
torkel@gaard:~$ 
```
### Qemu-guest-agent
```bash
torkel@gaard:~$ sudo apt-get install qemu-guest-agent
```

Issue `sudo shutdown now` to power of the guest and go to the Proxmox web gui and enable <kbd>QEMU Guest Agent</kbd> under Options, then start it again.

---

## Install Grafana
[https://grafana.com/docs/grafana/latest/installation/debian/](https://grafana.com/docs/grafana/latest/installation/debian/)

After the install, hop on in to `http://ip-adress:3000` and use <kbd>admin/admin</kbd> as login credentials. Then change the password for your user. For now, do not do anything in Grafana, but proceed with installation of `InfluxDB`. 

---

## Install InfluxDB
[https://docs.influxdata.com/influxdb/v1.8/introduction/install/](https://docs.influxdata.com/influxdb/v1.8/introduction/install/)
```bash
torkel@gaard:~$ sudo systemctl start influxdb
torkel@gaard:~$ sudo systemctl status influxdb
torkel@gaard:~$ sudo systemctl enable influxdb.service
```

Connect to InfluxDB and create a new database and two users. One user, which has **write** permissions (pfSense) and one user which has **read** permissions (Grafana).

```bash
torkel@gaard:~$ influx
Connected to http://localhost:8086 version 1.8.2
InfluxDB shell version: 1.8.2
> CREATE DATABASE "pf_firewall";
> CREATE USER "pf_firewall_write" WITH PASSWORD 'WRITE_PASSWORD';
> CREATE USER "pf_firewall_read" WITH PASSWORD 'READ_PASSWORD';
> GRANT READ ON pf_firewall TO pf_firewall_read
> GRANT WRITE ON pf_firewall TO pf_firewall_write
> exit
```

### Retention policy
How long do you want to keep this data? 

When you create a database, InfluxDB creates a retention policy called `autogen` with an infinite duration, a replication factor set to one, and a shard group duration set to seven days.

### Show current retention policy
```bash
torkel@gaard:~$ influx
Connected to http://localhost:8086 version 1.8.2
InfluxDB shell version: 1.8.2
> show databases
name: databases
name
----
_internal
pf_firewall
> show retention policies on pf_firewall
name    duration shardGroupDuration replicaN default
----    -------- ------------------ -------- -------
autogen 0s       168h0m0s           1        true
> 
```
### Create retention policy
```bash
> CREATE RETENTION POLICY 4weeks ON pf_firewall DURATION 4w REPLICATION 1 
> 
> show retention policies on pf_firewall
name     duration shardGroupDuration replicaN default
----     -------- ------------------ -------- -------
autogen  0s       168h0m0s           1        true
4weeks   672h0m0s 24h0m0s            1        false
```

### Alter retention policy ON database
```bash
> ALTER RETENTION POLICY twoweeks ON pf_firewall DURATION 4w REPLICATION 1 DEFAULT
>  
> show retention policies on pf_firewall
name     duration shardGroupDuration replicaN default
----     -------- ------------------ -------- -------
autogen  0s       168h0m0s           1        true
4weeks   336h0m0s 24h0m0s            1        false
```

---

## PfSense
### Install Telegraf
System `>` Package Manager `>` Available Packages, install `Telegraf`. 

### Plugins
Copy over all the plugins from [https://github.com/VictorRobellini/pfSense-Dashboard/tree/master/plugins](https://github.com/VictorRobellini/pfSense-Dashboard/tree/master/plugins) and place them in `/usr/local/bin` on your pfSense firewall. 

1. Enable SSH on your pfSense
2. Log in to pfSense through SSH
3. Change directory with `cd /usr/local/bin`, and then `fetch` the required files:
```bash
[2.4.5-RELEASE][clark@pfsense.arpa]/root: fetch https://github.com/VictorRobellini/pfSense-Dashboard/blob/master/plugins/telegraf_gateways-3.7.py
[2.4.5-RELEASE][clark@pfsense.arpa]/root: fetch https://github.com/VictorRobellini/pfSense-Dashboard/blob/master/plugins/telegraf_netifinfo_plugin
[2.4.5-RELEASE][clark@pfsense.arpa]/root: fetch https://github.com/VictorRobellini/pfSense-Dashboard/blob/master/plugins/telegraf_netifinfo_plugin.go
[2.4.5-RELEASE][clark@pfsense.arpa]/root: fetch https://github.com/VictorRobellini/pfSense-Dashboard/blob/master/plugins/telegraf_pfinterface.php
[2.4.5-RELEASE][clark@pfsense.arpa]/root: fetch https://github.com/VictorRobellini/pfSense-Dashboard/blob/master/plugins/telegraf_temperature.sh
[2.4.5-RELEASE][clark@pfsense.arpa]/root: fetch https://github.com/VictorRobellini/pfSense-Dashboard/blob/master/plugins/telegraf_unbound.sh
[2.4.5-RELEASE][clark@pfsense.arpa]/root: chmod 500 telegraf_*
```

### Configure Telegraf
Go to Services `>` Telegraf.

* Enable: <kbd>V</kbd> Enable Telegraf
* Telegraf Output: InfluxDB
* InfluxDB Server: http://ip-adress:8086
* InfluxDB Database: pf_firewall
* InfluxDB Username: pf_firewall_write
* InfluxDB Password: WRITE_PASSWORD

In the little config window on the bottom, paste in these lines of code:
```bash
[[inputs.exec]]
   commands = [
     "/usr/local/bin/telegraf_pfinterface.php",
     "/usr/local/bin/telegraf_gateways.py",
     "sh /usr/local/bin/telegraf_temperature.sh"
   ]
   data_format = "influx"

[[inputs.logparser]]
  files = ["/var/log/pfblockerng/dnsbl.log"]
  from_beginning=true
  [inputs.logparser.grok]
    measurement = "dnsbl_log"
    patterns = ["^%{WORD:BlockType}-%{WORD:BlockSubType},%{SYSLOGTIMESTAMP:timestamp:ts-syslog},%{IPORHOST:destination:tag},%{IPORHOST:source:tag},%{GREEDYDATA:call},%{WORD:BlockMethod},%{WORD:BlockList},%{IPORHOST:tld:tag},%{WORD:DefinedList:tag},%{GREEDYDATA:hitormiss}"]
    timezone = "Local"
    [inputs.logparser.tags]
      value = "1"

[[inputs.logparser]]
    files = ["/var/log/pfblockerng/ip_block.log"]
    from_beginning=true
    [inputs.logparser.grok]
        measurement = "ip_block_log"
        patterns = ["^%{SYSLOGTIMESTAMP:timestamp:ts-syslog},%{NUMBER:TrackerID},%{GREEDYDATA:Interface},%{WORD:InterfaceName},%{WORD:action},%{NUMBER:IPVersion},%{NUMBER:ProtocolID},%{GREEDYDATA:Protocol},%{IPORHOST:SrcIP:tag},%{IPORHOST:DstIP:tag},%{NUMBER:SrcPort},%{NUMBER:DstPort},%{WORD:Dir},%{WORD:GeoIP:tag},%{GREEDYDATA:AliasName},%{GREEDYDATA:IPEvaluated},%{GREEDYDATA:FeedName:tag},%{HOSTNAME:ResolvedHostname},%{HOSTNAME:ClientHostname},%{GREEDYDATA:ASN},%{GREEDYDATA:DuplicateEventStatus}"]
        timezone = "Local"

#[[inputs.unbound]]
#  server = "127.0.0.1:953"
#  binary = "/usr/local/bin/telegraf_unbound.sh"
```
Hit <kbd>Save</kbd>. 

---

## Configure Grafana
Configuraton `>` Data Sources 

* Add data source
* Select _InfluxDB_
* Name: **pf_firewall**
* URL: http://localhost:8086
* Database: pf_firewall
* User: pf_firewall_read
* Password: READ_PASSWORD
* HTTP Method: GET


### Grafana worldmap panel
```bash
torkel@gaard:~$ sudo grafana-cli plugins install grafana-worldmap-panel
```
### Grafana piechart-panel
```bash
torkel@gaard:~$ sudo grafana-cli plugins install grafana-piechart-panel
```

### Import pfSense-Grafana-Dashboard.json

[https://raw.githubusercontent.com/VictorRobellini/pfSense-Dashboard/master/pfSense-Grafana-Dashboard.json](https://raw.githubusercontent.com/VictorRobellini/pfSense-Dashboard/master/pfSense-Grafana-Dashboard.json)


Now you should be all done. Restart the `Telegraf` service on your pfSense firewall and the data should begin populating!

---

## TLS on Grafana
Do this if you run your own Certificate Authority and want to secure your dashboard. 
### Create Certificate Signing Request
```bash
torkel@gaard:/usr/local/etc/ssl$ cd ssl/
torkel@gaard:/usr/local/etc/ssl$ sudo openssl ecparam -name secp384r1 -out secp384r1.pem
torkel@gaard:/usr/local/etc/ssl$ sudo openssl ecparam -in secp384r1.pem -genkey -noout -out grafana.key
torkel@gaard:/usr/local/etc/ssl$ sudo openssl req -new -key grafana.key -out grafana.req
```
Send the request to your CA:
```bash
torkel@gaard:/usr/local/etc/ssl$ scp grafana.req ronald@shamir:/tmp/grafana.req
```

### Certificate Signing Request
#### -extfile
```bash
ronald@shamir:~$ cd ~/easy-rsa/config/
ronald@shamir:~/easy-rsa/config$ nano Grafana 
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = grafana
DNS.2 = 192.168.78.4
```
### Sign the CSR
```bash
ronald@shamir:~/easy-rsa/config$ openssl x509 -req -in /tmp/grafana.req  -CA ../pki/ca.crt -CAkey ../pki/private/ca.key -CAcreateserial -out grafana.crt -days 825 -sha512 -extfile ./Grafana 
Signature ok
subject=C = FR, ST = Paris, L = Paris, O = Boo, OU = IT, CN = 192.168.78.4, emailAddress = e-mail@mail.com
Getting CA Private Key
Enter pass phrase for ../pki/private/ca.key:
```
### Copy certificate back to Grafana
Copy `.crt` file to grafana
```bash
ronald@shamir:~/easy-rsa/config$ scp grafana.crt torkel@gaard:/tmp/grafana.crt
```
Move the file to an appropriate folder:
```bash
torkel@gaard:~$ mv /tmp/grafana.crt /usr/local/etc/ssl
```

### grafana.ini
Edit `grafana-server` to use the certificate and `https` protocol:
```bash
torkel@gaard:/usr/local/etc/ssl$ sudo nano /etc/grafana/grafana.ini 

[server]
# Protocol (http, https, socket)
protocol = https
(...)
# https certs & key file
cert_file = /usr/local/etc/ssl/grafana.crt
cert_key = /usr/local/etc/ssl/grafana.key
```

Restart `grafana-server`:
```bash
torkel@gaard:/usr/local/etc/ssl$ sudo systemctl restart grafana-server
```

## Fault finding
### Size of data
The size of the directory structure on disk will give the info for how large your database is:
```bash
torkel@gaard:/usr/local/etc/ssl$ du -sh /var/lib/influxdb/data/<db name>
```
Where `/var/lib/influxdb/data` is the data directory defined in `influxdb.conf`.

---

## Authors
Mr. Johnson

---

## Acknowledgments
* [https://stackoverflow.com/questions/44454836/influxdb-storage-size-on-disk](https://stackoverflow.com/questions/44454836/influxdb-storage-size-on-disk)
* [http://cactusprojects.com/setup-https-for-grafana/](http://cactusprojects.com/setup-https-for-grafana/)
* [https://towardsdatascience.com/influxdb-data-retention-f026496d708f](https://towardsdatascience.com/influxdb-data-retention-f026496d708f)
* [https://docs.influxdata.com/influxdb/v1.8/query_language/manage-database/#create-retention-policies-with-create-retention-policy](https://docs.influxdata.com/influxdb/v1.8/query_language/manage-database/#create-retention-policies-with-create-retention-policy)
* [https://docs.influxdata.com/influxdb/v1.8/query_language/manage-database/#create-retention-policies-with-create-retention-policy](https://docs.influxdata.com/influxdb/v1.8/query_language/manage-database/#create-retention-policies-with-create-retention-policy)
* [https://github.com/VictorRobellini/pfSense-Dashboard](https://github.com/VictorRobellini/pfSense-Dashboard)
* [https://grafana.com/grafana/plugins/grafana-worldmap-panel/installation](https://grafana.com/grafana/plugins/grafana-worldmap-panel/installation)
* [https://github.com/VictorRobellini/pfSense-Dashboard/tree/master/plugins](https://github.com/VictorRobellini/pfSense-Dashboard/tree/master/plugins)
* [https://chrisbergeron.com/2018/09/13/pfsense_grafana/](https://chrisbergeron.com/2018/09/13/pfsense_grafana/)
* [https://blog.lbdg.me/pfsense-monitoring/](https://blog.lbdg.me/pfsense-monitoring/)
* [http://www.andremiller.net/content/grafana-and-influxdb-quickstart-on-ubuntu](http://www.andremiller.net/content/grafana-and-influxdb-quickstart-on-ubuntu)



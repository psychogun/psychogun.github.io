---
layout: default
title: InfluxDBv2 dashboard for OPNsense
parent: OPNsense
nav_order: 2
---

# InfluxDBv2 dashboard for OPNsense
{: .no_toc }
This is how I used InfluxDB v2 to display dashboard for vitals from my OPNsense firewall.

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
* Ubuntu 20.04 LTS
* OPNsense 21.1.8_1-amd64
* InfluxDB 2.0.7
* Telegraf 1.19.0

---

## Timezone
Set your timezone:
```bash
paul@org-influxdbv2-01:~$ date
Sun Jul 25 22:58:59 UTC 2021

paul@org-influxdbv2-01:~$ sudo dpkg-reconfigure tzdata 
[sudo] password for graf: 

Current default time zone: 'Etc/UTC'
Local time is now:      Sun Jul 25 22:58:58 UTC 2021.
Universal Time is now:  Sun Jul 25 22:58:58 UTC 2021.

paul@org-influxdbv2-01:~$ date
Mon Jul 26 00:59:13 CEST 2021
```
---

## Install InfluxDB v2

* [https://docs.influxdata.com/influxdb/v2.0/install/](https://docs.influxdata.com/influxdb/v2.0/install/)


### InfluxDB v2 setup
```bash
root@org-influxdbv2-01:/home/paul# influx setup
Welcome to InfluxDB 2.0!
Please type your primary username: influx

Please type your password: 

Please type your password again: 

Please type your primary organization name: org

Please type your primary bucket name: org-opnsense-01

Please type your retention period in hours.
Or press ENTER for infinite: 72


You have entered:
  Username:          influx
  Organization:      org
  Bucket:            org-opnsense-01
  Retention Period:  72h0m0s
Confirm? (y/n): y

Config default has been stored in /root/.influxdbv2/configs.
User	Organization	Bucket
influx	org		org-opnsense-01
```

Log in to http://ip-address-of-influxdbv2:8086/

Go to Data, select "Buckets" and click +Add Data on your bucket.

---

## OPNsense configuration
OPNsense > System > Firmware > Plugins - search telegraf, install `os-telegraf`.

Go to Services > Telegraf and select Output. You'll need parameters for your bucket from your InfluxDBv2 installation. 

* Enable Influx v2 Output
* Influx V2 URL: Copy the "http://" section from section 3, "Start Telegraf" (http://172.22.15.10:8086)
* Influx v2 Token: Copy the token from "Configure your API Token" (HtvhPNu-P2WcIRch81X5dWJokgjZzDlLiAB6oxS_XpU24Wv9xu-lS3S4ht8_k5j6O_QSqdObCvSVap4JkU6noA==)
* Influx v2 Organization: org
* Influx v2 Bucket: org-opnsense-01

Click Save. 

Go to Services > Telegraf > General and 
* Enable Telegraf Agent

Click Save.

After a short while, Telegraf will send (default) data to your bucket; there are some system metrics that are enabled by default under the Input section in Telegraf. 

### Configure Intrusion Detection (Suricata)
* [https://www.influxdata.com/blog/network-security-monitoring-with-suricata-and-telegraf/](https://www.influxdata.com/blog/network-security-monitoring-with-suricata-and-telegraf/)

Check "Enable eve syslog output" under Intrusion Detection > Administration, hit Apply. 

Enabling this will create will create JSON entries in the file `/var/log/suricata/eve.json` whenever there is a new entry / Alert in Intrusion Detection.

### [[inputs.tail]]
To get Telegraf to grab these entries from the `eve.json` file and send it to your bucket in InfluxDBv2, you will have to enable "Intrusion Detection Alerts". 

Go to Services > Telegraf > Input and check 
* Intrusion Detection Alerts

This will create an entry in your `/usr/local/etc/telegraf.conf` file on the very bottom:
```bash
[[inputs.tail]]
  data_format = "json"
  files = ["/var/log/suricata/eve.json"]
  name_override = "suricata"
  tag_keys = ["event_type","src_ip","src_port","dest_ip","dest_port"]
  json_string_fields = ["*"]
```

Let us append some parameters to `[[inputs.tail]]` (`json_time_key` and `json_time_format`)- enable SSH to your OPNSense box.

Log in, and as root (`sudo su`), configure `[[inputs.tail]]`section to look like this:
```bash
vi /usr/local/etc/telegraf.conf 
[[inputs.tail]]
  data_format = "json"
  files = ["/var/log/suricata/eve.json"]
  name_override = "suricata-alerts"
  tag_keys = ["flow_id","in_iface","event_type","src_ip","src_port","dest_ip","dest_port","proto"]
  json_string_fields = ["*"]
  json_time_key = "timestamp"
  json_time_format = "2006-01-02T15:04:05-0700"
```

Then do a `service telegraf restart`, as root. To check if `telegraf` starts OK; `tail -f /var/log/telegraf/telegraf.log`.

* If you decide to change a setting / stop / start the Telegraf service from the GUI, you will notice that the `telegraf.conf` will be reset.

HOWEVER; 

Telegraf is started by user `telegraf`.
Intrusion Detection (Suricata) is started as `root`.

The Telegraf service does not have access rights to the file `/var/log/suricata/eve.json`.
```bash
sudo -u telegraf more /var/log/suricata/eve.json
/var/log/suricata/eve.json: Permission denied
```

Default permissions on the folder `/var/log/suricata` is `rwx------`, with user `root` as the owner and `wheel` as the group owner. 

Let us add `telegraf` to the wheel group:
```bash
root@opnsense:/usr/local/etc/suricata # pw group mod wheel -m telegraf
root@opnsense:/usr/local/etc/suricata # pw groupshow wheel
wheel:*:0:root,telegraf
```

Change permissions on the folder and the file `eve.json` to make the group able to read and execute:
```bash
chmod 750 /var/log/suricata
chmod 750 /var/log/suricata/eve.json
```
* This will not survive a reboot.

Now running `sudo -u telegraf more /var/log/suricata/eve.json` - you will see that the `telegraf` user is able to view the file. 

---

## [[inputs.suricata]]
There is an already built in plugin for Telegraf which you can use to monitor suricata with. It is called [[inputs.suricata]]. This plugin can take stats from suricata and produce them as metrics in InfluxDBv2. 


### Increase localhost buffer space
Under FreeBSD it is necessary to increase the localhost buffer space, otherwise messages from Suricata are truncated as they exceed the default available buffer space, consequently no statistics are processed by the [[inputs.suricata]] plugin.

How large this buffer space will have to be depends on your particular firewall. The number of interfaces you have matter in this regard, because the length of the event sent from Suricatat will increase due to this fact.

A method to figure out the correct size of the bufferspace, is to get suricata to write to a file by replacing `filetype : unix_stream` with `type : unix_stream` in `suricata.yaml`.

* You will have to stop telegraf (`service telegraf stop`) or use another filename in `suricata.yaml`:  `filename : /tmp/suricata-stats-file`.

After a while, you will see that suricata creates and writes to this `/tmp/suricata-stats-file` file. Copy one whole event, from {"timestamp:" to the next {"timestamp": (one event should end with }}}}), to a file.

If the file is for example 124230 bytes long, you should have the buffer space in BSD set to bit larger size than this.

Then you avoid getting truncated, and the [[inputs.suricata]] plugin will parse the event correctly.

Go to System > Settings > Tunables. Click +Add on the top right corner and add the 2 tunables:

* **Tunable**: net.local.stream.recvspace
* **Description**: Increase the localhost buffer space
* **Value**: 131072

* **Tunable**: net.local.stream.sendspace
* **Description**: Increase the localhost buffer space
* **Value**: 131072

Apply, then reboot the firewall. 

Verify with `sysctl -a | grep local.stream`:

```bash
sysctl -w net.local.stream.recvspace=131072
sysctl -w net.local.stream.sendspace=131072
```


### Configure the telegraf plugin
Suricata sends a stream of data to an already created UNIX-socket. Let us configure the [[inputs.suricatta]] plugin in Telegraf. 

Edit the telegraf.conf file on your OPNsense installation:
```bash
vi /usr/local/etc/telegraf.conf
```

Add to the bottom of the file:
```bash
[[inputs.suricata]]
  ## Data sink for Suricata stats log.
  # This is expected to be a filename of a
  # unix socket to be created for listening.
  source = "/tmp/run/suricata-stats.sock"

  # Delimiter for flattening field keys, e.g. subitem "alert" of "detect"
  # becomes "detect_alert" when delimiter is "_".
  delimiter = "_"

```

Restart the telegraf plugin; `service telegraf restart`.

### Configure eve-output
As of today's date, the [[inputs.suricata]] plugin in Telegraf shipped with OPNsense is not able to parse "alert" events. I found out this, because [[inputs.suricata]] was newly updated with the ability to do this, detect alerts; [https://github.com/influxdata/telegraf/tree/master/plugins/inputs/suricata](https://github.com/influxdata/telegraf/tree/master/plugins/inputs/suricata). Anyways, it is currently able to produce stats metrics. 

Add this configuration to the `custom.yaml` file, which is loaded by Suricata when you start `suricata` (I have thrown in alerts settings, just because):
```bash
cd /usr/local/etc/suricata
vi custom.yaml
  - eve-log:
      enabled: yes
      filetype: unix_stream
      filename: /tmp/suricata-stats.sock
      types:
        - stats:
           threads: yes
        - alert:
             # packet: yes              # enable dumping of packet (without stream segments)
             # metadata: no             # enable inclusion of app layer metadata with alert. Default yes
             # http-body: yes           # Requires metadata; enable dumping of http body in Base64
             # http-body-printable: yes # Requires metadata; enable dumping of http body in printable format

             # Enable the logging of tagged packets for rules using the
             # "tag" keyword.
             tagged-packets: yes

             http: yes
             tls: yes
```

* Remember, as this is YAML- you will have to use exact spaces to make the custom configuration parse correctly. 


Issuing `service suricata start` will make Suricata stream to this UNIX-socket `/tmp/suricata-stats.sock`. This eve-output can be in addition to the other default eve-output (`/var/log/suricata/eve.json`).

I have not found a way to check if there is any data flowing from Suricata to Telegraf, but using `sockstat | grep suricata` you can see there is a stream to this file from the service `suricata`.

After a while, stat metrics will show up in InfluxDBv2. 

* Remember to not use the GUI as the configuration / permissions on file / folder will reset.

---

## Firewall dashboard
...

---

## Fault finding
### Alert event
An alert event looks like this:
```bash
{"timestamp":"2021-07-23T19:04:13.108962+0200","flow_id":1155346044137668,"in_iface":"re1^","event_type":"anomaly","src_ip":"123.123.123.123","src_port":28967,"dest_ip":"231.231.231.231","dest_port":41398,"proto":"TCP","app_proto":"tls","anomaly":{"type":"applayer","event":"APPLAYER_DETECT_PROTOCOL_ONLY_ONE_DIRECTION","layer":"proto_detect"}}
```

### netcat
Does anything populate in the `/tmp/suricata-stats.sock` file?
```bash
root@opnsense:/usr/local/etc/suricata # nc -U /tmp/suricata-stats.sock 
```

`sockstat | grep suricata`

### Connection refused

Make sure the influxdb service is started:
```bash
E! [agent] Error writing to outputs.influxdb_v2: Post "http://172.22.15.10:8086/api/v2/telegrafs/07da6b3777610000/api/v2/write?bucket=org-opnsense-01&org=org": dial tcp 172.22.15.10:8086: connect: connection refused
```

### Wrong token
```bash
2021-07-18T13:45:30	 	E! [agent] Error writing to outputs.influxdb_v2: failed to write metric (401 Unauthorized): unauthorized: unauthorized access	 
2021-07-18T13:45:30	 	E! [outputs.influxdb_v2] When writing to [http://1.2.3.4:8086/api/v2/telegrafs/07da6c3776610000]: failed to write metric (401 Unauthorized): unauthorized: unauthorized access
```

Verify under InfluxDB, Data > Tokens.

### Telegraf bug

Bug Telegraf 1.19?
```bash
	 	time="2021-07-18T15:48:28+02:00" level=error msg="failed to open. Ignored. open /.cache/snowflake/ocsp_response_cache.json: no such file or directory\n" func="gosnowflake.(*defaultLogger).Errorf" file="log.go:120"	 
 	 	time="2021-07-18T15:48:28+02:00" level=error msg="failed to create cache directory. /.cache/snowflake, err: mkdir /.cache: permission denied. ignored\n" func="gosnowflake.(*defaultLogger).Errorf" file="log.go:120"
```

### Not truncated event
A whole stat event should start and end like this, before the new `{"timestamp":` shows up. 
```
{"timestamp":"2021-07-31T14:39:29.316063+0200","event_type":"stats","stats":{"uptime":144,"cap (...) muse":7169696}}}}}"
```

---

## Authors
Mr. Johnson

---


## Acknowledgments
* [https://github.com/influxdata/telegraf/issues/9330](https://github.com/influxdata/telegraf/issues/9330)
* [https://logz.io/blog/network-security-monitoring/](https://logz.io/blog/network-security-monitoring/)
* [https://buildmedia.readthedocs.org/media/pdf/suricata/latest/suricata.pdf](https://buildmedia.readthedocs.org/media/pdf/suricata/latest/suricata.pdf)
* [https://github.com/regit/suri-stats](https://github.com/regit/suri-stats)
* [https://grafana.com/grafana/dashboards/13384](https://grafana.com/grafana/dashboards/13384)
* [http://www.pfelk.com](http://www.pfelk.com)
* [https://fauie.com/2016/06/27/suricata-stats-to-influx-dbgrafana/](https://fauie.com/2016/06/27/suricata-stats-to-influx-dbgrafana/)
* [https://techviewleo.com/install-suricata-ids-ips-tool-on-rocky-linux/](https://techviewleo.com/install-suricata-ids-ips-tool-on-rocky-linux/)
* [https://forum.suricata.io/t/suricata-fails-to-create-socket/170](https://forum.suricata.io/t/suricata-fails-to-create-socket/170)
* [https://community.influxdata.com/t/custom-log-parsing-with-latest-tail-plugin-grok-and-influxdb-grafana-ready/14986](https://community.influxdata.com/t/custom-log-parsing-with-latest-tail-plugin-grok-and-influxdb-grafana-ready/14986)
* [https://forum.opnsense.org/index.php?topic=16966.0](https://forum.opnsense.org/index.php?topic=16966.0)
* [https://github.com/opnsense/core/issues/3401](https://github.com/opnsense/core/issues/3401)
* [https://community.influxdata.com/t/suricata-eve-json-input-file/16323](https://community.influxdata.com/t/suricata-eve-json-input-file/16323)
* [https://www.influxdata.com/blog/json-to-influxdb-with-telegraf-and-starlark/](https://www.influxdata.com/blog/json-to-influxdb-with-telegraf-and-starlark/)
* [https://alsargent.medium.com/how-we-use-influxdb-for-security-monitoring-7ade3973058e](https://alsargent.medium.com/how-we-use-influxdb-for-security-monitoring-7ade3973058e)
* [https://github.com/opnsense/plugins/issues/2239](https://github.com/opnsense/plugins/issues/2239)
* [https://www.reddit.com/r/OPNsenseFirewall/comments/7zrqhv/opnsense_telegraf_plugin/](https://www.reddit.com/r/OPNsenseFirewall/comments/7zrqhv/opnsense_telegraf_plugin/)
* [https://docs.influxdata.com/influxdb/v2.0/security/enable-tls/](https://docs.influxdata.com/influxdb/v2.0/security/enable-tls/)
* [https://github.com/tidwall/gjson#path-syntax](https://github.com/tidwall/gjson#path-syntax)
* [https://stackoverflow.com/questions/20234104/how-to-format-current-time-using-a-yyyymmddhhmmss-format/20234207#20234207](https://stackoverflow.com/questions/20234104/how-to-format-current-time-using-a-yyyymmddhhmmss-format/20234207#20234207)
* [https://play.golang.org/p/6WuZaOy3OiX](https://play.golang.org/p/6WuZaOy3OiX)
* [https://stackoverflow.com/questions/285658/run-as-different-user-under-freebsd](https://stackoverflow.com/questions/285658/run-as-different-user-under-freebsd)
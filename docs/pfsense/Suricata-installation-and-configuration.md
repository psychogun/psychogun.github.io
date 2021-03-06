---
layout: default
title: Suricata installation and configuration
parent: pfSense
nav_order: 6
---
# Suricata installation and configuration
{: .no_toc }
What is the only reason for not running Snort? If you are using **Suricata** instead. 

This is how I installed `Suricata` and used it as a IDS/IPS on my pfSense firewall and logged events to my Elastic Stack.

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
Install the Suricata package by navigating to <kbd>System</kbd>, <kbd>Package Manager</kbd> and select <kbd>Available Packages</kbd>.

### Prerequisites
* pfSense 2.4.4-RELEASE-p3 (amd64)
* suricata 4.1.6_2
* elastic stack 5.6.8

---

## Configuration
Navigate to Suricata by clicking <kbd>Services</kbd>, <kbd>Suricata</kbd>.

### Global Settings
#### Please Choose The Type Of Rules You Wish To Download
Install ETOpen Emerging Threats rules: 
- [x] ETOpen is a free open source set of Suricata rules whose coverage is more limited than ETPro.

Install Snort rules: 
- [x] Snort free Registered User or paid Subscriber rules
* Snort Rules Filename: **snortrules-snapshot-29151.tar.gz**
* Snort Oinkmaster Code: **d3fb58191764f206a2a444buid8fd289sd891z6c**

Install Snort GPLv2 Community rules: 
- [x] The Snort Community Ruleset is a GPLv2 Talos-certified ruleset that is distributed free of charge without any Snort Subscriber License restrictions.<br>
Hide Deprecated Rules Categories: 
- [x] Hide deprecated rules categories in the GUI and remove them from the configuration. Default is Not Checked.

#### Rules Update Settings
* Update Interval: 12 HOURS
* Update Start Time: 00:30
* GeoLite2 DB Update:
- [x] Enable downloading of free GeoLite2 Country IP Database updates. Default is Not Checked
* GeoLite2 DB License Key: dOFdszB21sz44

#### General Settings
* Remove Blocked Hosts Interval: NEVER
* Log to System Log: [x] Copy Suricata messages to the firewall system log.
* Keep Suricata Settings After Deinstall: [v] Settings will not be removed during package deinstallation.

Click <kbd>Save</kbd>. 

### Updates
No rule sets have been updated. Click <kbd>Update</kbd>. After you have configured the above settings in <kbd>Global Settings</kbd>, it should read <kbd>Results: success</kbd>.
#### INSTALLED RULE SET MD5 SIGNATURES
* Emerging Threats Open Rules
* Snort Subscriber Rules
* Snort GPLv2 Community Rules

### Interfaces
On the Interface Setting Overview, click + Add and all the way to the bottom, click Save.

Go back to Interfaces and click the blue icon "Start suricata on this interface". Edit that WAN interface.
#### Logging Settings
Send Alerts to System Log: 
- [x] Suricata will send Alerts from this interface to the firewall's system log.
* Log Facility: LOCAL1
* Log Priority: NOTICE

Enable Tracked-Files Log:
- [x] Suricata will log tracked files in JavaScript Object Notation (JSON) format. Default is Not Checked.<br>
Append Tracked-Files Log: 
- [x] Suricata will append-to instead of clearing Tracked Files log file when restarting. Default is Checked.

#### EVE Output Settings
- EVE JSON Log: 
- [x] Suricata will output selected info in JSON format to a single file or to syslog. Default is Not Checked.<br>
- EVE Output Type: <kbd>SYSLOG</kbd> 

Let the rest be default, click <kbd>Save</kbd>. 

---

## 10-suricata.conf
```bash
root@ELK:/usr/local/etc/logstash/conf.d # nano 12-suricata.conf
filter {
  if [type] == "SuricataIDPS" {
    date {
      match => [ "timestamp", "ISO8601" ]
    }
    ruby {
      code => "if event['event_type'] == 'fileinfo'; event['fileinfo']['type']=event['fileinfo']['magic'].to_s.split(',')[0]; end;"
    }
  }

  if [src_ip]  {
    geoip {
      source => "src_ip"
      target => "geoip"
      database => "/usr/local/etc/logstash/GeoIP/GeoLite2-City.mmdb"
      add_field => [ "[geoip][coordinates]", "%{[geoip][longitude]}" ]
      add_field => [ "[geoip][coordinates]", "%{[geoip][latitude]}"  ]
    }
    mutate {
      convert => [ "[geoip][coordinates]", "float" ]
    }
    if ![geoip.ip] {
      if [dest_ip]  {
        geoip {
          source => "dest_ip"
          target => "geoip"
          database => "/usr/local/etc/logstash/GeoIP/GeoLite2-City.mmdb"
          add_field => [ "[geoip][coordinates]", "%{[geoip][longitude]}" ]
          add_field => [ "[geoip][coordinates]", "%{[geoip][latitude]}"  ]
        }
        mutate {
          convert => [ "[geoip][coordinates]", "float" ]
        }
      }
    }
  }
}
```
---

## Authors
Mr. Johnson

---

## Acknowledgments
* [http://pfelksuricata.3ilson.com](http://pfelksuricata.3ilson.com)
* [http://pfsensesetup.com/sitemap.xml](http://pfsensesetup.com/sitemap.xml) (Search for suricata)
* [https://forum.netgate.com/topic/70170/taming-the-beasts-aka-suricata-blueprint/13](https://forum.netgate.com/topic/70170/taming-the-beasts-aka-suricata-blueprint/13)
* [https://suricata-ids.org](https://suricata-ids.org)
* [https://cybersecurity.att.com/blogs/security-essentials/open-source-intrusion-detection-tools-a-quick-overview](https://cybersecurity.att.com/blogs/security-essentials/open-source-intrusion-detection-tools-a-quick-overview)

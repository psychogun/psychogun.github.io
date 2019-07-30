---
layout: default
title: How to install Elastic Stack on Ubuntu 18.04 LTS and use it with pfSense 2.4.4
parent: Linux
nav_order: 1
---

# How to install Elastic Stack on Ubuntu 18.04 LTS and use it with PFSense 2.4.4
{: .no_toc }
So, on a whim I googled syslog + pfsense, and I saw some images of some nice dashboards (Kibana) for the firewall logs from PFSense. The tutorials I found did not tell me exactly how this all works, particularly how Elasticsearch, Logstash and Kibana work together. 

This is how i installed the Elastic Stack (Elasticsearch, Logstash, Kibana, Beats and SHIELD) on a Ubuntu 18.04 with encrypted communication, so that I could have a nice visualization of my PFSense firewall logs (syslog) and a flow chart of my network (NetFlow).

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}
---
## Getting started
Logstash combines logdata from different sources to a joint Java Script Object Notation (JSON)-format. Elasticsearch indexes and saves JSON-logdata in a central database. Kibana graphically presents logdata to the user on a webbrowser. 

### Logstash
Logstash continiously looks after (different types of) logdata which is presented (syslog on TCP/UDP, or FileBeat, or Netflow, ++) on specified ports.

Logstash accepts different types of traffic, applies a filter and transforms the logdata to JSON-format which is then sent to Elasticsearch for indexing and saving in a central database.

### Elasticsearch
Elasticsearch is a search- and analyze tool based on Apache Lucene. Elasticsearch uses Schema-free database-structure and all data is saved in a JSON-format. You can communicate with Elasticsearch' database directly be sending requests in JSON-format. This makes Elasticsearch flexible, because the communication is not locked to Kibana. 

### Kibana
Kibana presents logdata to the user in graphical webpages. The webpages can be used for system monitoring and can be updated in real-time. Kibana is configurable so that the user can choose which data will be extracted from the Elasticsearch database and how they will be presented. 

In Kibana you can create customized control panels for various applications. Kibana combined with SHIELD controls access to the control panel according to the need-to-know principle. 

### SHIELD
SHIELD contains a security API that lays on the top of the ELK stack and acts as a security layer. 

SHIELD has the following main features: 
* Authentication
* Authorization
* Secure network communication
* IP filtering
* Traceability

SHIELD is a commcercial product, so you'll have to pay for it. 

### PFSense
pfSense is an open source firewall/router computer software distribution based on FreeBSD. It is installed on a physical computer or a virtual machine to make a dedicated firewall/router for a network. It can be configured and upgraded through a web-based interface, and requires no knowledge of the underlying FreeBSD system to manage.

## Disclaimer
I am using Github for my own sake and configuration files. Use at own risk. And with every how-to's, read through the entire thing before starting.  

### Prerequisites
* Internet connection?
* Clean Ubuntu 18.04 LTS install
* Java
* Elasticsearch 
* Logstash
* Kibana
* PFSene 2.4.4

## Installlation
Firstmost, update and upgrade our Ubuntu installation:
```
elk@stack:~$ sudo apt-get update
elk@stack:~$ sudo apt-get upgrade
```
### Installing Java
Logstash requires Java 8 or Java 11, so we'll go ahead and install the OpenJDK Runtime Environment:
```
elk@stack:~$  sudo apt install -y openjdk-8-jdk
```
Was it installed properly?
```
elk@stack:~$ java -version
```
You'll have to ensure that your JAVA HOME environment is properly set up. To see your current JAVA HOME environment variable, issue command:
```
elk@stack:~$ echo $JAVA_HOME 
```
If nothing shows up, your JAVA_HOME environment path was not set. To set your JAVA_HOME path, run:
```
elk@stack:~$ export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
```
Add JAVA bin directory to the PATH variable:
```
elk@stack:~$ export PATH=$PATH:$JAVA_HOME/bin
```
Check your PATH variable:
```
elk@stack:~$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/usr/lib/jvm/java-8-openjdk-amd64/bin
```

## Installing Elasticsearch
We'll install Elasticsearch first, because Logstash and Kibana are dependent on Elasticsearch (atleast I read that somewhere). When you install each component of the stack, the install will automatically create service users for running the different components. 

Import the Elasticsearch PGP key and install from the APT repository by following this excellent guide; 
* https://www.elastic.co/guide/en/elasticsearch/reference/current/deb.html (remember to install wget and apt-transport-https)

In order for Elasticsearch to function optimally, you should increase the number of files the (service) user 'elasticsearch' is allowed to open at the same time. Edit ```limits.conf``` and add the following:
```
elk@stack:~$ nano /etc/security/limits.conf 
elasticsearch soft nofile 32000
elasticsearch hard nofile 32000
```
Also edit /etc/sysctl.conf and add the following:
```
fs.file-max = 300000
```
For some more information about file handling, read; https://gist.github.com/luckydev/b2a6ebe793aeacf50ff15331fb3b519d

To start Elasticsearch, write:
```
elk@stack:~$ sudo systemctl start elasticsearch
```
To enable Elasticsearch to start when you boot, write:
```
elkeson@elk:~$ sudo systemctl enable elasticsearch
```
To see if Elasticsearch has started, curl it:
```
curl -X GET http://localhost:9200
```
If everything installed as it should, the output of curl will look something like this:
```
elk@stack:~$ curl -X GET http://localhost:9200
{
  "name" : "stack",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "kEi4W1ffRcqYhab1ato3xQ",
  "version" : {
    "number" : "7.2.0",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "508c38a",
    "build_date" : "2019-06-20T15:54:18.811730Z",
    "build_snapshot" : false,
    "lucene_version" : "8.0.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}

```
## Installing Logstash
After Elasticsearch is installed, we'll go ahead and install Logstash.

Follow this excellent guide in in order to install the Public Signing Key and save the repository definition so you can 'sudo apt-get install logstash' (APT):
* https://www.elastic.co/guide/en/logstash/current/installing-logstash.html

After installing the package, you can enable and start up Logstash with:
```
elk@stack:~$ sudo systemctl enable logstash.service
elk@stack:~$ sudo systemctl start logstash.service
```
To see if our Logstash service is running as it should, issue command:
```
elk@stack:~$ sudo systemctl status logstash.service
```
And to stop it?
```
elk@stack:~$ sudo systemctl stop logstash.service
```
Just stop it for now. 

### Pipelines
So.  Pipelines. 
In order to specify how Logstash listens to incoming connections (ports/protocols) and where to send data (elasticsearch), and whether or not to apply a filter, we will have to create configuration file(s) in the JSON-format. Where to put these configuration files ("pipelines"), are stated in ```pipelines.yml```;
```
elk@stack:~$ more /etc/logstash/pipelines.yml 
# This file is where you define your pipelines. You can define multiple.
# For more information on multiple pipelines, see the documentation:
#   https://www.elastic.co/guide/en/logstash/current/multiple-pipelines.html

- pipeline.id: main
  path.config: "/etc/logstash/conf.d/*.conf"
```
You can have one 'big' .conf configuration file, or you can separate them in multiple .conf configuration files. I have them separated:

Input file: ```/etc/logstash/conf.d/01-inputs.conf```
```
elk@stack:~$ more /etc/logstash/conf.d/01-inputs.conf 
input {  
  tcp {
    type => "syslog"
    port => 5120
  }
}

input {  
  udp {
    type => "syslog"
    port => 5140
  }
}
```
The input file listens for syslog's on both TCP and UDP at port 5120 and 5140. 


Syslog filter file: ```/etc/logstash/conf.d/10-syslog.conf```
```
elk@stack:~$ more /etc/logstash/conf.d/10-syslog.conf 
filter {
  if [type] == "syslog" {
    if [host] =~ /192\.168\.40\.1/ {
      mutate {
        add_tag => ["pfsense", "Ready"]
      }
    }
    if [host] =~ /172\.2\.22\.1/ {
      mutate {
        add_tag => ["pfsense-2", "Ready"]
      }
    }

    if "Ready" not in [tags] {
      mutate {
        add_tag => [ "syslog" ]
      }
    }
  }
}
filter {
  if [type] == "syslog" {
    mutate {
      remove_tag => "Ready"
    }
  }
}
filter {
  if "syslog" in [tags] {
    grok {
      match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}" }
      add_field => [ "received_at", "%{@timestamp}" ]
      add_field => [ "received_from", "%{host}" ]
    }
    syslog_pri { }
    date {
      match => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM  dd HH:mm:ss" ]
      locale => "en"
    }
    if !("_grokparsefailure" in [tags]) {
      mutate {
        replace => [ "@source_host", "%{syslog_hostname}" ]
        replace => [ "@message", "%{syslog_message}" ]
      }
    }
    mutate {
      remove_field => [ "syslog_hostname", "syslog_message", "syslog_timestamp" ]
    }
  }
}
```
I will not try to explain what the filter for syslog exactly does, because I have no experience with JSON. But you can see that it tags syslog traffic from my PFSense with both 'pfsense' and 'Ready', and adds some extra fields. The if [host] above (192.168.40.1) is the IP adress to my PFSense 2.4.4 firewall, which address you'll probably have to change. And if you have another pfsense box, add it here as well (the one with 172.22.2.1 address).

PFSense filter: ```/etc/logstash/conf.d/11-pfsense.conf```
```
elk@stack:~$ more /etc/logstash/conf.d/11-pfsense.conf 
filter {
  if "pfsense" in [tags] {
    grok {
      add_tag => [ "firewall" ]
      match => [ "message", "<(?<evtid>.*)>(?<datetime>(?:Jan(?:uary)?|Feb(?:ruary)?|Mar(?:ch)?|Apr(?:il)?|May|Jun(?:e)?|Jul(?:y)?|Aug(?:ust)?|Sep(?:tember)?|Oct(?:ober)?|Nov(?:ember)?|Dec(?:ember)?)\s+(?:(?:0[1-9])|(?:[12][0-9])|(?:3[01])|[1-9]) (?:2[0123]|[01]?[0-9]):(?:[0-5][0-9]):(?:[0-5][0-9])) (?<prog>.*?): (?<msg>.*)" ]
            }
            mutate {
              gsub => ["datetime","  "," "]
            }
            date {
              match => [ "datetime", "MMM dd HH:mm:ss" ]
              timezone => "Europe/Paris"
            }
            mutate {
              replace => [ "message", "%{msg}" ]
            }
            mutate {
              remove_field => [ "msg", "datetime" ]
            }
            if [prog] =~ /^dhcpd$/ {
              mutate {
                add_tag => [ "dhcpd" ]
              }
             grok {
                 patterns_dir => ["/etc/logstash/conf.d/patterns"]
                 match => [ "message", "%{DHCPD}"]
             }
            }
            if [prog] =~ /^suricata/ {
              mutate {
                add_tag => [ "SuricataIDPS" ]
              }
              grok {
                  patterns_dir => ["/etc/logstash/conf.d/patterns"]
                  match => [ "message", "%{PFSENSE_SURICATA}"]
             }
                if ![geoip] and [ids_src_ip] !~ /^(10\.|192\.168\.)/ {
                  geoip {
                    add_tag => [ "GeoIP" ]
                    source => "ids_src_ip"
                    database => "/etc/logstash/GeoLite2-City.mmdb"
                  }
                }
            if [prog] =~ /^suricata/ {
                mutate {
                    add_tag => [ "ET-Sig" ]
                    add_field => [ "Signature_Info", "http://doc.emergingthreats.net/bin/view/Main/%{[ids_sig_id]}" ]
                }
              }
            }
            if [prog] =~ /^charon$/ {
              mutate {
                add_tag => [ "ipsec" ]
              }
            }
            if [prog] =~ /^barnyard2/ {
              mutate {
                add_tag => [ "barnyard2" ]
              }
            }
            if [prog] =~ /^openvpn/ {
              mutate {
                add_tag => [ "openvpn" ]
              }
            }
            if [prog] =~ /^ntpd/ {
              mutate {
                add_tag => [ "ntpd" ]
              }
            }
            if [prog] =~ /^php-fpm/ {
              mutate {
                add_tag => [ "web_portal" ]
              }
              grok {
                  patterns_dir => ["/etc/logstash/conf.d/patterns"]
                  match => [ "message", "%{PFSENSE_APP}%{PFSENSE_APP_DATA}"]
              }
              mutate {
                  lowercase => [ 'pfsense_ACTION' ]
              }
            }
            if [prog] =~ /^apinger/ {
              mutate {
                add_tag => [ "apinger" ]
              }
            }
            if [prog] =~ /^filterlog$/ {
                mutate {
                    remove_field => [ "msg", "datetime" ]
                }
                grok {
                    add_tag => [ "firewall" ]
                    patterns_dir => ["/etc/logstash/conf.d/patterns"]
                    match => [ "message", "%{PFSENSE_LOG_DATA}%{PFSENSE_IP_SPECIFIC_DATA}%{PFSENSE_IP_DATA}%{PFSENSE_PROTOCOL_DATA}",
                               "message", "%{PFSENSE_IPv4_SPECIFIC_DATA}%{PFSENSE_IP_DATA}%{PFSENSE_PROTOCOL_DATA}",
                               "message", "%{PFSENSE_IPv6_SPECIFIC_DATA}%{PFSENSE_IP_DATA}%{PFSENSE_PROTOCOL_DATA}"]
                }
                mutate {
                    lowercase => [ 'proto' ]
                }
                if ![geoip] and [src_ip] !~ /^(10\.|192\.168\.)/ {
                  geoip {
                    add_tag => [ "GeoIP" ]
                    source => "src_ip"
                    database => "/etc/logstash/GeoLite2-City.mmdb"
                  }
                }
            }
        }
}
```

Since we have the Elastic stack installed on the same machine, our Logstash would connect to Elasticsearch (30-outputs.conf) like this:
```
elkn@stack:~$ more /etc/logstash/conf.d/30-outputs.conf 
output {
        elasticsearch {
                hosts => ["http://localhost:9200"]
                index => "logstash-%{+YYYY.MM.dd}" }
}
```
The index statement dictates that a rotating of the database file with a date stamp will occour. If you want to debug the Logstash configuration, you could also send output to console by adding stdout:
```
elkeson@elk:~$ more /etc/logstash/conf.d/30-outputs.conf 
output {
        elasticsearch {
                hosts => ["http://localhost:9200"]
                index => "logstash-%{+YYYY.MM.dd}" }
                stdout { codec => rubydebug} 
}
```
Or just use tail -f and see what the output is whenever you are starting logstash. 
```
elk@stack:/etc/logstash/conf.d$ tail -f /var/log/logstash/logstash-plain.log 
```
Now, if you haven't already stopped logstash, do it. Because we are referencing geoip and a different non-standard grok pattern in our logstash configs, so logstash would not boot well. 

### Download and install MaxMind GeoIP database
```
elk@stack:~$ cd /etc/logstash
elk@stack:~$ sudo wget http://geolite.maxmind.com/download/geoip/database/GeoLite2-City.mmdb.gz
elk@stack:~$ sudo gunzip GeoLite2-City.mmdb.gz
```

### Grok
Grok is a great way to parse unstructured log data into something structured and queryable. Sometimes logstash doesn’t have a pattern you need, so you'll have to make it yourself. Or download it. From here, or wherever I first found it.
```bash
elkn@stack:~$ cd /etc/logstash/conf.d
elkn@stack:~$ sudo mkdir patterns
elkn@stack:~$ sudo nano pfsense_2_4_2.grok
# GROK Custom Patterns (add to patterns directory and reference in GROK filter for pfSense events):
# GROK Patterns for pfSense 2.4.2 Logging Format
#
# Created 27 Jan 2015 by J. Pisano (Handles TCP, UDP, and ICMP log entries)
# Edited 14 Feb 2015 by Elijah Paul elijah.paul@gmail.com
# Edited 10 Mar 2015 by Bernd Zeimetz <bernd@bzed.de>
# Edited 28 Oct 2017 by Brian Turek <brian.turek@gmail.com>
# Edited 5 Jan 2017 by Andrew Wilson <andrew@3ilson.com>
# Edited 30 Apr 2019 by Mike Eriksson <mike@swedishmike.org>
# taken from https://gist.github.com/elijahpaul/3d80030ac3e8138848b5
#
# - Adjusted IPv4 to accept pfSense 2.4.2
# - Adjusted IPv6 to accept pfSense 2.4.2
#
# TODO: Add/expand support for IPv6 messages.

PFSENSE_LOG_ENTRY %{PFSENSE_LOG_DATA}%{PFSENSE_IP_SPECIFIC_DATA}%{PFSENSE_IP_DATA}%{PFSENSE_PROTOCOL_DATA}?
PFSENSE_LOG_DATA %{INT:rule},%{INT:sub_rule}?,,%{INT:tracker},%{DATA:iface},%{WORD:reason},%{WORD:action},%{WORD:direction},
PFSENSE_IP_DATA %{INT:length},%{IP:src_ip},%{IP:dest_ip},
PFSENSE_IP_SPECIFIC_DATA %{PFSENSE_IPv4_SPECIFIC_DATA}|%{PFSENSE_IPv6_SPECIFIC_DATA}
PFSENSE_IPv4_SPECIFIC_DATA (?<ip_ver>(4)),%{BASE16NUM:tos},%{WORD:ecn}?,%{INT:ttl},%{INT:id},%{INT:offset},%{WORD:flags},%{INT:proto_id},%{WORD:proto},
PFSENSE_IPv6_SPECIFIC_DATA (?<ip_ver>(6)),%{BASE16NUM:IPv6_Flag1},%{WORD:IPv6_Flag2},%{WORD:flow_label},%{WORD:proto_type},%{INT:proto_id},
PFSENSE_PROTOCOL_DATA %{PFSENSE_UDP_DATA}|%{PFSENSE_TCP_DATA}|%{PFSENSE_ICMP_DATA}|%{PFSENSE_IGMP_DATA}|%{PFSENSE_IPv6_VAR}
PFSENSE_UDP_DATA %{INT:src_port},%{INT:dest_port},%{INT:data_length}
PFSENSE_TCP_DATA %{INT:src_port},%{INT:dest_port},%{INT:data_length},%{WORD:tcp_flags},%{INT:sequence_number},%{INT:ack_number},%{INT:tcp_window},%{DATA:urg_data},%{GREEDYDATA:tcp_options}
PFSENSE_IGMP_DATA datalength=%{INT:data_length}
PFSENSE_ICMP_DATA %{PFSENSE_ICMP_TYPE}%{PFSENSE_ICMP_RESPONSE}
PFSENSE_ICMP_TYPE (?<icmp_type>(request|reply|unreachproto|unreachport|unreach|timeexceed|paramprob|redirect|maskreply|needfrag|tstamp|tstampreply)),
PFSENSE_ICMP_RESPONSE %{PFSENSE_ICMP_ECHO_REQ_REPLY}|%{PFSENSE_ICMP_UNREACHPORT}| %{PFSENSE_ICMP_UNREACHPROTO}|%{PFSENSE_ICMP_UNREACHABLE}|%{PFSENSE_ICMP_NEED_FLAG}|%{PFSENSE_ICMP_TSTAMP}|%{PFSENSE_ICMP_TSTAMP_REPLY}
PFSENSE_ICMP_ECHO_REQ_REPLY %{INT:icmp_echo_id},%{INT:icmp_echo_sequence}
PFSENSE_ICMP_UNREACHPORT %{IP:icmp_unreachport_dest_ip},%{WORD:icmp_unreachport_protocol},%{INT:icmp_unreachport_port}
PFSENSE_ICMP_UNREACHPROTO %{IP:icmp_unreach_dest_ip},%{WORD:icmp_unreachproto_protocol}
PFSENSE_ICMP_UNREACHABLE %{GREEDYDATA:icmp_unreachable}
PFSENSE_ICMP_NEED_FLAG %{IP:icmp_need_flag_ip},%{INT:icmp_need_flag_mtu}
PFSENSE_ICMP_TSTAMP %{INT:icmp_tstamp_id},%{INT:icmp_tstamp_sequence}
PFSENSE_ICMP_TSTAMP_REPLY %{INT:icmp_tstamp_reply_id},%{INT:icmp_tstamp_reply_sequence},%{INT:icmp_tstamp_reply_otime},%{INT:icmp_tstamp_reply_rtime},%{INT:icmp_tstamp_reply_ttime}

PFSENSE_IPv6_VAR %{WORD:Type},%{WORD:Option},%{WORD:Flags},%{WORD:Flags}

# DHCP (Optional)
DHCPD (%{DHCPDISCOVER}|%{DHCPOFFER}|%{DHCPREQUEST}|%{DHCPACK}|%{DHCPINFORM}|%{DHCPRELEASE})
DHCPDISCOVER %{WORD:dhcp_action} from %{COMMONMAC:dhcp_client_mac}%{SPACE}(\(%{GREEDYDATA:dhcp_client_hostname}\))? via (?<dhcp_client_vlan>[0-9a-z_]*)(: %{GREEDYDATA:dhcp_load_balance})?
DHCPOFFER %{WORD:dhcp_action} on %{IPV4:dhcp_client_ip} to %{COMMONMAC:dhcp_client_mac}%{SPACE}(\(%{GREEDYDATA:dhcp_client_hostname}\))? via (?<dhcp_client_vlan>[0-9a-z_]*)
DHCPREQUEST %{WORD:dhcp_action} for %{IPV4:dhcp_client_ip}%{SPACE}(\(%{IPV4:dhcp_ip_unknown}\))? from %{COMMONMAC:dhcp_client_mac}%{SPACE}(\(%{GREEDYDATA:dhcp_client_hostname}\))? via (?<dhcp_client_vlan>[0-9a-z_]*)(: %{GREEDYDATA:dhcp_request_message})?
DHCPACK %{WORD:dhcp_action} on %{IPV4:dhcp_client_ip} to %{COMMONMAC:dhcp_client_mac}%{SPACE}(\(%{GREEDYDATA:dhcp_client_hostname}\))? via (?<dhcp_client_vlan>[0-9a-z_]*)
DHCPINFORM %{WORD:dhcp_action} from %{IPV4:dhcp_client_ip} via %(?<dhcp_client_vlan>[0-9a-z_]*)
DHCPRELEASE %{WORD:dhcp_action} of %{IPV4:dhcp_client_ip} from %{COMMONMAC:dhcp_client_mac}%{SPACE}(\(%{GREEDYDATA:dhcp_client_hostname}\))? via

# PFSENSE
PFSENSE_CARP_DATA (%{WORD:carp_type}),(%{INT:carp_ttl}),(%{INT:carp_vhid}),(%{INT:carp_version}),(%{INT:carp_advbase}),(%{INT:carp_advskew})
PFSENSE_APP (%{DATA:pfsense_APP}),
PFSENSE_APP_DATA (%{PFSENSE_APP_LOGOUT}|%{PFSENSE_APP_LOGIN}|%{PFSENSE_APP_ERROR}|%{PFSENSE_APP_GEN})
PFSENSE_APP_LOGIN (%{DATA:pfsense_ACTION}) for user \'(%{DATA:pfsense_USER})\' from: (%{GREEDYDATA:pfsense_REMOTE_IP})
PFSENSE_APP_LOGOUT User (%{DATA:pfsense_ACTION}) for user \'(%{DATA:pfsense_USER})\' from: (%{GREEDYDATA:pfsense_REMOTE_IP})
PFSENSE_APP_ERROR webConfigurator (%{DATA:pfsense_ACTION}) for \'(%{DATA:pfsense_USER})\' from (%{GREEDYDATA:pfsense_REMOTE_IP})
PFSENSE_APP_GEN (%{GREEDYDATA:pfsense_ACTION})

# SURICATA
PFSENSE_SURICATA %{SPACE}\[%{NUMBER:ids_gen_id}:%{NUMBER:ids_sig_id}:%{NUMBER:ids_sig_rev}\]%{SPACE}%{GREEDYDATA:ids_desc}%{SPACE}\[Classification:%{SPACE}%{GREEDYDATA:ids_class}\]%{SPACE}\[Priority:%{SPACE}%{NUMBER:ids_pri}\]%{SPACE}{%{WORD:ids_proto}}%{SPACE}%{IP:ids_src_ip}:%{NUMBER:ids_src_port}%{SPACE}->%{SPACE}%{IP:ids_dest_ip}:%{NUMBER:ids_dest_port}
```



Now, go ahead and start logstash!
```
elkn@stack:~$ sudo systemctl start logstash
elkn@stack:~$ sudo tail -f /var/log/logstash/logstash-plain.log 
```

## Installing Kibana 
Follow this excellent guide to install Kibana and make it start automatically when the system boots :
* https://www.elastic.co/guide/en/kibana/current/deb.html

Kibana loads its configuration from the /etc/kibana/kibana.yml file by default. The format of this config file is explained in https://www.elastic.co/guide/en/kibana/7.2/settings.html

If our Elasticsearch database is not on the same host as Kibana, you will have to tell where Elasticsearch is by specifying 'elasticsearch.hosts', e.g.: elasticsearch.hosts http://ip-adress:9200 in kibana.yml

What we first and foremost want to do with our kibana.yml, is to edit the ```server.host: "localhost"```and bind it to (all) interfaces by writing ```server.host: "0.0.0.0"```. Editing this parameter from a non-loopback address enables connections from remote users.

Restart (or start) your Kibana and go to http://ip-adress:5601 to check if it is up and running (choose No when asked if you want to import some data. Select 'Explore on your own', we'll get back to Kibana in a bit. Now we need some data to visualize, eg make PFSense send data to logstash.

## Configuring PFSense for syslog
Log on to your PFSense and go to Status > System logs > Settings. 

For content, we will for now log 'Firewall Events'.

Enable Remote Logging and point one of the 'Remote log servers' to 'logstash-syslog-input-ip:and-port', e.g.: 192.168.4.100:5140. Syslog sends UDP datagrams to port 514 on the specified remote syslog server, unless another port is specified.

## Index patterns, discovers, dashboards and visualizations
Index patterns tell Kibana which Elasticsearch indices you want to explore. An index pattern can match the name of a single index, or include a wildcard (*) to match multiple indices.

For example, Logstash typically creates a series of indices in the format logstash-YYYY.MMM.DD. To explore all of the log data from May 2018, you could specify the index pattern logstash-2018.05*.

Discover enables you to explore your data with Kibana’s data discovery functions. You have access to every document in every index that matches the selected index pattern. You can submit search queries, filter the search results, and view document data. Go to Discover to see your syslogs flowing in!

Kibana visualizations are based on Elasticsearch queries. By using a series of Elasticsearch aggregations to extract and process your data, you can create charts that show you the trends, spikes, and dips you need to know about.

A Kibana dashboard displays a collection of visualizations, searches, and maps. You can arrange, resize, and edit the dashboard content and then save the dashboard so you can share it.

Go to http://ip-adress:5601 and go to Management > Create Index Pattern (Kibana Index Patterns) > and our logstash service which we started have enabled us to select that indicies, so write "logstash*". Press 'Next step'.  Under 'Time Filter field name' choose '@timestamp' and then hit 'Create Index pattern'. 

With 'Saved Objects' you are able to import searches, dashboards and visualizations that has been made before. Let us do that.

Go to Saved Objects > Import > and import 'Discover - Firewall and pfSense.json' and you might have to re-associate the object with your logstash* index pattern. You are successfull when you have imported 6 objects. 

Now import 'Visualizations - Firewall and pfSense.json', and you might have to re-associate the object with your logstash* index pattern. You are successfull when you have imported 31 objects.

Now you can create your own dashboard and mix those visualizations in how you please. But, someone has luckily done this for us before ;) Import your chosen 'Dashboard - ******.json' dashboard files! 


## Configuring PFSense for NetFlow
Log on to your PFSense and go to System > Package Manager > Available Packages and install ```softflowd```. Edit ```softlowd```by navigating to Services > softlowd. A basic configuration looks like this:

* Select which interfaces to monitor. I selected WAN.
* Enter your ELK server IP address for Host.
* Enter 2056 for Port.
* Select Netflow version 9.
* Set Flow Tracking Level to Full.
* Click Save.

Communication between PFSense and Logstash for Netflow is not encrypted. So make sure you are creating a good network design by using VLAN or something else to ensure your metadata of the communication on the monitored interface is not intentionally going where it should not go. 

### Configure Logstash for NetFlow
Stop logstash.service:
```
elk@stack:/usr/share/logstash$ sudo systemctl stop logstash.service
```

For a first time use, run logstash with netflow and the --setup parameter:
```
elk@stack:/usr/share/logstash$ sudo /usr/share/logstash/bin/logstash --modules netflow --setup 
```
The --setup parameter will add additional Dashboards and vizualisations on your Kibana dashboard. If you run with parameter ```--setup``` one more time, it will override your Dashboard edits. So run the above command only once. Let it run for some time before you Ctrl + C out of it (1-2 minutes?).

What I discovered, was that, logstash is ignoring the 'pipelines.yml' file because modules or command line options are specified. So what we want to do, is to edit the '/usr/share/logstash/modules/netflow/configuration/logstash/netflow.conf.erb' file to a conf file, like the ones you now have in '/etc/logstash/conf.d/'. 
```
elk@stack:~$ cd /usr/share/logstash/modules/netflow/configuration/logstash
elk@stack:/usr/share/logstash/modules/netflow/configuration/logstash$  cp netflow.conf.erb /etc/logstash/conf.d/netflow.conf
```
The config is an ERB template so there are sections that are overwritten, in between <%=, %>, with real values - replace these and you will have a netflow config. 
```
elk@stack:/usr/share/logstash$ sudo nano netflow.conf
```


```
elk@stack:/usr/share/logstash$ sudo systemctl start logstash
```
Ignoring the 'pipelines.yml' file because modules or command line options are specified


Our final logstash.service will boot with these parameters: (edit the ```logstash.service``` file and add --modules;



### Enable HTTPS on Kibana
You are able to access the Kibana interface via HTTPS by setting ```server.sssl.enable```to true in ```kibana.yml``` configuration file.To be able to do so, you have to create your certificates.

Generate a key with a pass-phrase:
```
elkeson@elk:~$ cd /etc/kibana
elkeson@elk:/etc/kibana$ sudo openssl genrsa -des3 -out server.key 2048
```
Now create the insecure key, the one without a passphrase, and shuffle the key names:
```
elkeson@elk:/etc/kibana$ sudo openssl rsa -in server.key -out server.key.insecure
elkeson@elk:/etc/kibana$ sudo mv server.key server.key.secure
elkeson@elk:/etc/kibana$ sudo mv server.key.insecure server.key
```
The insecure key is now named server.key, and you can use this file to generate the CSR without passphrase. CSR stands for Certificate Signing Request. 

To create the CSR, run the following command at a terminal prompt:
```
elk@stack:/etc/kibana$ sudo openssl req -new -key server.key -out server.csr
```
This ```server.csr```file would normally go to a Certificate Authoriy (CA) for issuing a 'stamp' saying; hey- this is a legit site. Instead of sending this to a CA, we are making this 'stamp' our selves by creating a Self-Signed Certificate:
```
elk@stack:/etc/kibana$ sudo openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
```
Now you have server.crt and server.key. 

In ```kibana.yml``` we point to these certifications:

```
# Enables SSL and paths to the PEM-format SSL certificate and SSL key files, respectively.
# These settings enable SSL for outgoing requests from the Kibana server to the browser.
server.ssl.enabled: true
server.ssl.certificate: /etc/kibana/server.crt
server.ssl.key: /etc/kibana/server.key
```
Make server.key readable for the service user kibana;
```
elk@stack:/etc/kibana$ sudo chmod 644 server.key
```
Stop and start kibana:
```
elk@stack:/etc/kibana$ sudo systemctl stop kibana.service 
elk@stack:/etc/kibana$ sudo systemctl start kibana.service 
```
Now you are able to communicate with Kibana over an encrypted connection (https).



```
elk@stack:/usr/share/logstash$ sudo nano /etc/systemd/system/logstash.service 
ExecStart=/usr/share/logstash/bin/logstash "--path.settings" "/etc/logstash" --modules netflow
```
Restart dameon for making edits take effect:
```
elk@stack:/usr/share/logstash$ sudo systemctl daemon-reload 
```
Start logstash:
```
elk@stack:/usr/share/logstash$ sudo systemctl start logstash.service 
```
Check status of logstash:
```
elk@stack:/usr/share/logstash$ sudo systemctl status logstash.service 
```
Now you are able to go in Kibana > Dashboard > and select different Dashboards provided from Kibana for different NetFlow charts!


## Configure Kibana to connect to Elasticsearch via HTTPS
Even though we are using the Open Source version of Kibana, we are able to encrypt communication between Kibana and Elasticsearch. For further hardening of your Elastic Stack, go to https://www.elastic.co/subscriptions to see if you are willing to pay the dues. 

Add the ```xpack.security.enabled: true``` in elasticsearch.yml 



## HTTPS on Kibana
Up until now, your communication with the Kibana interface has been on a HTTP connection. 


openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt


openssl genrsa -des3 -out server.key 2048
openssl rsa -in server.key -out server.key
openssl req -sha256 -new -key server.key -out server.csr -subj '/CN=localhost'
openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
Replace 'localhost' with your domain name. Run commands one by one because openssl will prompt you same values for certificate generation

Edit ```logstash.service```:
```
elk@stack:/usr/share/logstash$ sudo nano /etc/systemd/system/logstash.service
```
Add  ```--modules netflow --setup -M netflow.var.input.udp.port=2055```:
``` 
ExecStart=/usr/share/logstash/bin/logstash "--path.settings" "/etc/logstash" --modules netflow --setup -M netflow.var.input.udp.port=2055
```
elkeson@elk:/usr/share/logstash$ systemctl daemon-reload
elkeson@elk:/usr/share/logstash$ systemctl start logstash.service

Change directory to logstash installation directory:
```elk@stack:/$ cd /usr/share/logstash/```

Run this command to install / enable the netflow module in logstash:
```
elkeson@elk:/usr/share/logstash$ bin/logstash --modules netflow --setup -M netflow.var.input.udp.port=2055
```

For further information, go check out the excellent guide here: https://www.elastic.co/guide/en/logstash/current/netflow-module.html


### Yellow cluster
http://chrissimpson.co.uk/elasticsearch-yellow-cluster-status-explained.html


#### Version
To see which version of the program you have installed are, do:
```
elkeson@elk:~$ /usr/share/logstash/bin/logstash --version
logstash 6.8.1
```

Elastic produce a full range of log shippers known as ‘Beats’ which run as lightweight agents on the source devices and transmit data to a destination either running Elasticsearch or Logstash. If you are using Beats you can do this to make it use SSL to encrypt the communication between the Beat agent and Logstash:

```
elkeson@elk:/etc/ssl$ sudo openssl req -x509 -nodes -newkey rsa:2048 -days 365 -keyout logstash-forwarder.key -out logstash-forwarder.crt -subj /CN=elk.h37b.local

sudo openssl pkcs8 -in logstash-forwarder.key  -topk8 -nocrypt -out logstash-forwarder.key.pem

sudo chmod 644 /etc/ssl/logstash-forwarder.key.pem
```

```
elkeson@elk:/etc/logstash$ more pipelines.yml 
# This file is where you define your pipelines. You can define multiple.
# For more information on multiple pipelines, see the documentation:
#   https://www.elastic.co/guide/en/logstash/current/multiple-pipelines.html

- pipeline.id: main
  path.config: "/etc/logstash/conf.d/*.conf"
```  
  
```
sudo nano /etc/logstash/conf.d/logstash.conf

input {
 beats {
   port => 5044
   
   # Set to False if you do not SSL
   ssl => true
  
   # Delete below lines if no SSL is used
   ssl_certificate => "/etc/ssl/logstash-forwarder.crt"
   ssl_key => "/etc/ssl/logstash-forwarder.key.pem"
   }
}
filter {
if [type] == "syslog" {
    grok {
      match => { "message" => "%{SYSLOGLINE}" }
    }

    date {
match => [ "timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
}
  }

}
output {
 elasticsearch {
  hosts => localhost
    index => "%{[@metadata][beat]}-%{+YYYY.MM.dd}"
       }
stdout {
    codec => rubydebug
       }
}
```

```
sudo systemctl restart logstash
sudo systemctl enable logstash
```
sudo cat /var/log/logstash/logstash-plain.log

sudo apt install -y kibana

Configure your kibana server;
sudo nano /etc/kibana/kibana.yml


server.host: "0.0.0.0"
elasticsearch.hosts: "http://localhost:9200"

elasticsearch.url: "http://localhost:9200"

sudo systemctl restart kibana
sudo systemctl enable kibana

Logg inn på localhost:5601, click Create Index. 

Management > Saved Objects > 




PS: This is a fun read-through; https://www.elastic.co/elk-stack 



Edit ```logstash.yml```:
```
elk@stack:/usr/share/logstash$ sudo nano /etc/logstash/logstash.yml 

(...)
modules:
   - name: netflow
     var.input.udp.port: 2056
     var.elasticsearch.hosts: http://127.0.0.1:9200
     var.elasticsearch.ssl.enabled: false
     var.kibana.host: 127.0.0.1:5601
     var.kibana.scheme: http
     var.kibana.ssl.enabled: false
     var.kibana.ssl.verification_mode: disable
    
```

### Mentions (in no particular order)
* https://arnaudloos.com/2019/enable-x-pack-security/
* https://extelligenceblog.it/2017/10/18/elasticstack-elk-and-pfsense-firewall-ip-traffic-statistics-with-netflow/
* https://help.ubuntu.com/lts/serverguide/certificates-and-security.html
* https://github.com/solvaholic/solvaholic.github.io/wiki/Netflow-with-pfSense-and-ELK
* https://www.elastic.co/elk-stack
* https://github.com/patrickjennings/logstash-pfsense
* https://www.itzgeek.com/how-tos/linux/ubuntu-how-tos/how-to-install-elasticsearch-logstash-and-kibana-elk-stack-on-ubuntu-18-04-ubuntu-16-04.html
* https://logz.io/blog/elk-stack-raspberry-pi/
* https://www.itzgeek.com/how-tos/linux/ubuntu-how-tos/how-to-install-elasticsearch-logstash-and-kibana-elk-stack-on-ubuntu-18-04-ubuntu-16-04.html
* http://extelligenceblog.it/2017/07/11/elastic-stack-suricata-idps-and-pfsense-firewall-part-1/
* https://forum.netgate.com/topic/107735/elk-pfsense-2-3-working/2
* http://pfelk.3ilson.com
* https://vitux.com/how-to-setup-java_home-path-in-ubuntu/
* http://secretwafflelabs.com/2015/11/06/pfsense-elk/
* https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html#plugins-filters-grok-patterns_dir

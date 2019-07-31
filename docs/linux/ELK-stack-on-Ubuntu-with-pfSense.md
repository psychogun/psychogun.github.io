---
layout: default
title: ELK Stack on Ubuntu with pfSense
parent: Linux
nav_order: 1
---
*WORK IN PROGRESS as of 2019-07-31*
# How to install the ELK Stack on Ubuntu for pfSense
{: .no_toc }
So, on a whim I googled syslog + pfsense, and I saw some images of some nice dashboards (Kibana) for the firewall logs from PFSense. The tutorials I found did not tell me exactly how this all works, particularly how Elasticsearch, Logstash and Kibana work together. 


These instructions will tell you what I have learned and how I installed the Elastic Stack (Elasticsearch, Logstash, Kibana, Beats and SHIELD) on Ubuntu with encrypted communication, so that I could have a nice visualization of my PFSense firewall logs with syslogs and netflow.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}
---
## Getting started
Logstash combines logdata from different sources to a joint Java Script Object Notation (JSON)-format. Elasticsearch indexes and saves JSON-logdata in a central database. Kibana graphically presents logdata to the user in a webbrowser. 

### Logstash
Logstash continiously looks after (different types of) logdata which is presented (syslog, or FileBeat, or Netflow, ++) on specified ports.

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

SHIELD is a commercial product, so you'll have to pay for it. 

### pfSense
pfSense is an open source firewall/router computer software distribution based on FreeBSD. It is installed on a physical computer or a virtual machine to make a dedicated firewall/router for a network. It can be configured and upgraded through a web-based interface, and requires no knowledge of the underlying FreeBSD system to manage.

## Disclaimer
As with every how-to's, read through the entire thing before starting. 

### Prerequisites
* Clean Ubuntu 18.04 LTS 
* Java 8
* Elasticsearch 7.2 
* Logstash 7.2
* Kibana 7.2
* PFSene 2.4.4

## Installlation
First and foremost, update and upgrade our Ubuntu installation:
```
elk@stack:~$ sudo apt-get update
elk@stack:~$ sudo apt-get upgrade
```
### Installing Java
Logstash requires Java 8 or Java 11, so we'll go ahead and install the OpenJDK Runtime Environment. I have used Java 8:
```bash
elk@stack:~$  sudo apt install -y openjdk-8-jdk
```
Was it installed properly?
```bash
elk@stack:~$ java -version
openjdk version "1.8.0_212"
OpenJDK Runtime Environment (build 1.8.0_212-8u212-b03-0ubuntu1.18.04.1-b03)
OpenJDK 64-Bit Server VM (build 25.212-b03, mixed mode)
```
You'll have to ensure that your JAVA HOME environment is properly set up. To see your current JAVA HOME environment variable, issue command:
```bash
elk@stack:~$ echo $JAVA_HOME 
```
If nothing shows up, your JAVA_HOME environment path is not set. To set your JAVA_HOME path, run:
```bash
elk@stack:~$ export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
```
Add JAVA bin directory to the PATH variable:
```bash
elk@stack:~$ export PATH=$PATH:$JAVA_HOME/bin
```
Check your PATH variable:
```bash
elk@stack:~$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/usr/lib/jvm/java-8-openjdk-amd64/bin
```

## Installing Elasticsearch
We'll install Elasticsearch first, because Logstash and Kibana are dependent on Elasticsearch. When you install each component of the stack, the install will automatically create service users for running the different components. 

Import the Elasticsearch PGP key and install Elasticsearch from the APT repository by following this excellent guide; 
* [https://www.elastic.co/guide/en/elasticsearch/reference/current/deb.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/deb.html) (also make sure to install `wget` and `apt-transport-https`).

In order for Elasticsearch to function optimally, you should increase the number of files the (service) user 'elasticsearch' is allowed to open at the same time. Edit `limits.conf` and add the following:
```bash
elk@stack:~$ nano /etc/security/limits.conf 
elasticsearch soft nofile 32000
elasticsearch hard nofile 32000
```
Also edit /etc/sysctl.conf and add the following:
```bash
fs.file-max = 300000
```
For some more information about file handling, read [https://gist.github.com/luckydev/b2a6ebe793aeacf50ff15331fb3b519d](https://gist.github.com/luckydev/b2a6ebe793aeacf50ff15331fb3b519d).

To start Elasticsearch, write:
```bash
elk@stack:~$ sudo systemctl start elasticsearch
```
To enable Elasticsearch to start when you boot, write:
```bash
elkeson@elk:~$ sudo systemctl enable elasticsearch
```
To see if Elasticsearch has been started as it should, your curl output will look something like this:
```bash
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

Follow this excellent guide in in order to install the Public Signing Key and save the repository definition so you can `sudo apt-get install logstash` (APT):
* [https://www.elastic.co/guide/en/logstash/current/installing-logstash.html](https://www.elastic.co/guide/en/logstash/current/installing-logstash.html)

After installing the package, you can enable and start up Logstash with:
```bash
elk@stack:~$ sudo systemctl enable logstash.service
elk@stack:~$ sudo systemctl start logstash.service
```
To see if our Logstash service is running as it should, issue command:
```bash
elk@stack:~$ sudo systemctl status logstash.service
```
And to stop it?
```bash
elk@stack:~$ sudo systemctl stop logstash.service
```
Just stop it for now. 

### Pipelines
So.  Pipelines. 
In order to specify how Logstash listens to incoming connections (ports/protocols) and where to send data (elasticsearch), and whether or not to apply a filter, we will have to create configuration files (input, output, filter) in the JSON-format. Where to put these configuration files ("pipelines"), are stated in `pipelines.yml`:
```bash
elk@stack:~$ more /etc/logstash/pipelines.yml 
# This file is where you define your pipelines. You can define multiple.
# For more information on multiple pipelines, see the documentation:
#   https://www.elastic.co/guide/en/logstash/current/multiple-pipelines.html

- pipeline.id: main
  path.config: "/etc/logstash/conf.d/*.conf"
```
You can have one 'big' .conf configuration file, or you can separate them in multiple .conf configuration files. You will see I have some separated, and one joined:

Input file: `/etc/logstash/conf.d/01-inputs.conf`
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
This input file listens for syslog's on both TCP and UDP at port 5120 and 5140. 


Syslog filter file: `/etc/logstash/conf.d/10-syslog.conf`
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
I will not try to explain what the filter for syslog exactly does, because I have no experience with JSON. But you can see that it tags syslog traffic from my pfSense with both 'pfsense' and 'Ready', and adds some extra fields. The if [host] above (192.168.40.1) is the IP adress to my pfSense firewall, which address you'll probably have to change. And if you have another pfSense firewall, add it here as well (the one with 172.22.2.1 address).

pfSense filter: `/etc/logstash/conf.d/11-pfsense.conf`
```bash
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
```bash
elkn@stack:~$ more /etc/logstash/conf.d/30-outputs.conf 
output {
        elasticsearch {
                hosts => ["http://localhost:9200"]
                index => "logstash-%{+YYYY.MM.dd}" }
}
```
The index statement dictates that a rotating of the database file with a date stamp will occour. If you want to view the Logstash output realtime, you could also send output to console by adding stdout:
```bash
elkeson@elk:~$ more /etc/logstash/conf.d/30-outputs.conf 
output {
        elasticsearch {
                hosts => ["http://localhost:9200"]
                index => "logstash-%{+YYYY.MM.dd}" }
                stdout { codec => rubydebug } 
}
```
**Disclaimer: I have to investigate why that is not working (stdout)

Use `tail -f` to see the status of the logstash service: 
```bash
elk@stack:/etc/logstash/conf.d$ tail -f /var/log/logstash/logstash-plain.log 
```
If you have tried to start logstash, you can now stop it by `sudo systemctl stop logstash.service`. Because we are referencing geoip and a different non-standard grok pattern in our logstash configs, which we haven't installed yet, you'll see through logstash-plain.log that the logstash will not boot well.  

### Download and install MaxMind GeoIP database
```bash
elk@stack:~$ cd /etc/logstash
elk@stack:~$ sudo wget http://geolite.maxmind.com/download/geoip/database/GeoLite2-City.mmdb.gz
elk@stack:~$ sudo gunzip GeoLite2-City.mmdb.gz
```

### Grok
Grok is a great way to parse unstructured log data into something structured and queryable. Sometimes logstash doesn’t have a pattern you need, so you'll have to make it yourself. Or download it.
```bash
elk@stack:~$ cd /etc/logstash/conf.d/
elk@stack:~$ sudo mkdir patterns
elk@stack:~$ cd patterns
elk@stack:~$ sudo wget https://raw.githubusercontent.com/a3ilson/pfelk/master/pfsense_2_4_2.grok
```

Now, go ahead and start logstash!
```bash
elkn@stack:~$ sudo systemctl start logstash
```
Hopefully it started succesfully:
```bash
elkn@stack:~$ sudo tail -f /var/log/logstash/logstash-plain.log 
```

## Installing Kibana 
Follow this excellent guide to install Kibana and make it start automatically when the system boots :
* [https://www.elastic.co/guide/en/kibana/current/deb.html](https://www.elastic.co/guide/en/kibana/current/deb.html)

Kibana loads its configuration from the /etc/kibana/kibana.yml file by default. The format of this config file is explained in [https://www.elastic.co/guide/en/kibana/7.2/settings.html](https://www.elastic.co/guide/en/kibana/7.2/settings.html)

If our Elasticsearch database is not on the same host as Kibana, you will have to tell where Elasticsearch is by specifying 'elasticsearch.hosts', e.g.: elasticsearch.hosts http://ip-adress:9200 in kibana.yml

What we first and foremost want to do with our kibana.yml, is to edit the `server.host: "localhost"`and bind it to (all) interfaces by writing `server.host: "0.0.0.0"`. Editing this parameter from a non-loopback address enables connections from remote users, thus enables us to connect to Kibana through a web-browser.

Restart (or start) your Kibana with `sudo systemctl start kibana` and go to http://ip-adress:5601 to check if it is up and running (choose No when asked if you want to import some data). Select 'Explore on your own', we'll get back to Kibana in a bit. Now we need some data to visualize, e.g. make pfSense send data to logstash.

## Configuring pfSense for syslog
Log on to your pfSense and go to Status > System logs > Settings. 

For content, we will for now log 'Firewall Events'.

Enable Remote Logging and point one of the 'Remote log servers' to 'logstash-syslog-input-ip:and-port', e.g.: 192.168.4.100:5140, as stated in `01-inputs.conf`. Syslog sends UDP datagrams to port 514 on the specified remote syslog server, unless another port is specified.

## Index patterns, discovers, dashboards and visualizations
*_Index patterns_* tell Kibana which Elasticsearch indices you want to explore. An index pattern can match the name of a single index, or include a wildcard (*) to match multiple indices.

For example, Logstash typically creates a series of indices in the format logstash-YYYY.MMM.DD. To explore all of the log data from May 2018, you could specify the index pattern logstash-2018.05*.

*_Discover_* enables you to explore your data with Kibana’s data discovery functions. You have access to every document in every index that matches the selected index pattern. You can submit search queries, filter the search results, and view document data. Go to Discover to see your syslogs flowing in!

Kibana *_visualizations_* are based on Elasticsearch queries. By using a series of Elasticsearch aggregations to extract and process your data, you can create charts that show you the trends, spikes, and dips you need to know about.

A Kibana *_dashboard_* displays a collection of visualizations, searches, and maps. You can arrange, resize, and edit the dashboard content and then save the dashboard so you can share it.

Go to http://ip-adress:5601 and go to Management > Create Index Pattern (Kibana Index Patterns) > and our logstash service which we started have enabled us to select that indicies, so write "logstash*". Press 'Next step'.  Under 'Time Filter field name' choose '@timestamp' and then hit 'Create Index pattern'. 

With 'Saved Objects' you are able to import searches, dashboards and visualizations that has been made before. Let us do that.

Go to Saved Objects > Import > and import 'Discover - Firewall and pfSense.json' and you might have to re-associate the object with your logstash* index pattern. You are successfull when you have imported 6 objects. 

Now import 'Visualizations - Firewall and pfSense.json', and you might have to re-associate the object with your logstash* index pattern. You are successfull when you have imported 31 objects.

Now you can create your own dashboard and mix those visualizations in how you please. But, someone has luckily done this for us before ;) Import your chosen 'Dashboard - ******.json' dashboard files! 


## Configuring PFSense for NetFlow
Log on to your PFSense and go to System > Package Manager > Available Packages and install `softflowd`. Edit `softlowd` by navigating to Services > softlowd. A basic configuration looks like this:

* Select which interfaces to monitor. I selected WAN.
* Enter your ELK server IP address for Host.
* Enter 2055 for Port.
* Select Netflow version 9.
* Set Flow Tracking Level to Full.
* Click Save.

Communication between PFSense and Logstash for Netflow is not encrypted. So make sure you are creating a good network design by using VLAN or something else to ensure your metadata of the communication on the monitored interface is not intentionally going where it should not go. 

### Configure Logstash for NetFlow
Stop logstash.service:
```bash
elk@stack:/usr/share/logstash$ sudo systemctl stop logstash.service
```

For a first time use, run logstash with netflow and the --setup parameter:
```bash
elk@stack:/usr/share/logstash$ sudo /usr/share/logstash/bin/logstash --modules netflow --setup 
```
The --setup parameter will add additional Dashboards and vizualisations on your Kibana dashboard. If you run with parameter `--setup` one more time, it will override your (eventual) netflow dashboard edits. So run the above command only once. Let it run for some time before you Ctrl + C out of it (1-2 minutes?).

What I discovered when specifying netflow as a module in logstash.yml, was that _logstash is ignoring the 'pipelines.yml' file because modules or command line options are specified_. So the pfSense firewall logs broke.

So what we want to do, is to edit the `/usr/share/logstash/modules/netflow/configuration/logstash/netflow.conf.erb` file to a conf file, like the ones you now have in `/etc/logstash/conf.d/`. 
```bash
elk@stack:~$ cd /usr/share/logstash/modules/netflow/configuration/logstash
elk@stack:/usr/share/logstash/modules/netflow/configuration/logstash$  cp netflow.conf.erb /etc/logstash/conf.d/netflow.conf
```
The config is an ERB template so there are sections that are overwritten, in between <%=, %>, with real values - replace these and you will have a netflow config. 
```bash
elk@stack:/usr/share/logstash$ cd conf.d/
elk@stack:/usr/share/logstash$ sudo wget 
```

```bash
elk@stack:/usr/share/logstash$ sudo systemctl start logstash
```



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

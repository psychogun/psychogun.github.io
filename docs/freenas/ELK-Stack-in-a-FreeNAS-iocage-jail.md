---
layout: default
title: How to install the Elastic Stack in a FreeNAS iocage jail
parent: FreeNAS
nav_order: 5
---

# How to install the Elastic Stack in a FreeNAS iocage jail
{: .no_toc }
I have had some troubles with my Virtualized Ubuntu installation running the Elastic Stack, so I thought that it might be better having the Elastic Stack run in a jail instead. Unfortunately, the  Elastic Stack found through the packet manager is version 6.8.5, whilst on Linux it currently on version 7.5.1 (as of 2019-12-30). 
I do now know what new features there are on the newer versions. 

If you are new to Elastic Stack, I suggest reading [https://psychogun.github.io/docs/linux/ELK-stack-on-Ubuntu-with-pfSense/](https://psychogun.github.io/docs/linux/ELK-stack-on-Ubuntu-with-pfSense/) where I have elaborated some more on this topic.

This is how I installed the Elastic Stack in a FreeNAS iocage jail with syslog, netflow and suricata. 

## Table of contents
{: .no_toc .text-delta }
1. TOC
{:toc}
---

## Getting started
These instructions will install and configure the Elastic stack in an iocage jail on FreeNAS for use with pfSense. A lot of this is copy and paste from the excellent tutorial by [http://blog.dushin.net/2019/08/installing-elk-on-freenas-jail/](http://blog.dushin.net/2019/08/installing-elk-on-freenas-jail/) and [http://pfelksuricata.3ilson.com](http://pfelksuricata.3ilson.com).

### Prerequisites
* FreeNAS 11.3
* elasticsearch6: 6.8.5
* logstash6: 6.8.5
* kibana6: 6.8.5
* suricata 4.1.6_2


## Create Jail
Go to FreeNAS gui and select Jails and click ADD.

Press ADVANCED JAIL CREATION

Name: ELK
Release: 11.3-RELEASSE
v DHCP Autoconfigure IPv4
v VNET
v Berkeley Packet Filter

In order to run logstash, we need access to the /proc filesystem. This will require that the allow_mount_procfs jail property be set, which in turn requires that the allow_mount jail property be set, and that the enforce_statfs jail property is set to something less than 2. We set this property to 1.

Go to Basic Properties and
`v Auto-start``

Go to Jail Properties and 
`enforce statfs` is set to `1`.
`v allow_mount`
`v allow_mount_procfs`

fdescfs is required by Java, which will be installed later:
`v mount_fdescfs`

You could also do this manually by editing `etc/fstab`, but we'll do it in the GUI instead:
Custom Properties:
`v mount_procfs`

Then click SAVE.

### Attach to Jail
```bash
root@freenas:~ # iocage list
+-----+----------------+-------+--------------+------+
| 90  | ELK            | up    | 11.3-RELEASE | DHCP |
+-----+----------------+-------+--------------+------+
root@freenas:~ # iocage console ELK
FreeBSD 11.3-STABLE (FreeNAS.amd64) #0 r325575+c9231c7d6bd(HEAD): Mon Nov 18 22:46:47 UTC 2019

Welcome to FreeBSD!

Release Notes, Errata: https://www.FreeBSD.org/releases/
Security Advisories:   https://www.FreeBSD.org/security/
FreeBSD Handbook:      https://www.FreeBSD.org/handbook/
FreeBSD FAQ:           https://www.FreeBSD.org/faq/
Questions List: https://lists.FreeBSD.org/mailman/listinfo/freebsd-questions/
FreeBSD Forums:        https://forums.FreeBSD.org/

Documents installed with the system are in the /usr/local/share/doc/freebsd/
directory, or can be installed later with:  pkg install en-freebsd-doc
For other languages, replace "en" with a language code like de or fr.

Show the version of FreeBSD installed:  freebsd-version ; uname -a
Please include that output and any error messages when posting questions.
Introduction to manual pages:  man man
FreeBSD directory layout:      man hier

Edit /etc/motd to change this login announcement.
root@ELK:~ # 
```

### pkg and nano
```bash
root@ELK:~ # pkg install nano
```

### Change from quarterly to latest
To receive the latest updates from the packet manager, change this:
```bash
root@ELK:~ # nano /etc/pkg/FreeBSD.conf 
```
Change
```bash
FreeBSD: {
  url: "pkg+http://pkg.FreeBSD.org/${ABI}/quarterly",
```
to
```bash
FreeBSD: {
  url: "pkg+http://pkg.FreeBSD.org/${ABI}/latest",
```

### upgrade
```bash
root@ELK:~ # pkg upgrade
Updating FreeBSD repository catalogue...
pkg: Repository FreeBSD has a wrong packagesite, need to re-create database
[ELK] Fetching meta.txz: 100%    944 B   0.9kB/s    00:01    
[ELK] Fetching packagesite.txz: 100%    6 MiB   1.1MB/s    00:06    
Processing entries: 100%
FreeBSD repository update completed. 31647 packages processed.
All repositories are up to date.
Checking for upgrades (2 candidates): 100%
Processing candidates (2 candidates): 100%
The following 1 package(s) will be affected (of 0 checked):

Installed packages to be UPGRADED:
	nano: 4.4 -> 4.6

Number of packages to be upgraded: 1

521 KiB to be downloaded.

Proceed with this action? [y/N]: y
[ELK] [1/1] Fetching nano-4.6.txz: 100%  521 KiB 266.9kB/s    00:02    
Checking integrity... done (0 conflicting)
[ELK] [1/1] Upgrading nano from 4.4 to 4.6...
[ELK] [1/1] Extracting nano-4.6: 100%

root@ELK:~ # 
```

## Install and Start Elasticsearch
Elasticsearch is the (Java) service that hosts all of the Lucene injestion, indexing, and search capability.

Installation of Elasticsearch 6.8.5:

```bash
root@ELK:~ # pkg install elasticsearch6
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
The following 30 package(s) will be affected (of 0 checked):

New packages to be INSTALLED:
	elasticsearch6: 6.8.5
	bash: 5.0.11
	openjdk8: 8.232.09.1_1
	libXtst: 1.2.3_2
	libXi: 1.7.10,1
	libXfixes: 5.0.3_2
	libX11: 1.6.9,1
	libxcb: 1.13.1
	libXdmcp: 1.1.3
	xorgproto: 2019.2
	libXau: 1.0.9
	libxml2: 2.9.10
	libpthread-stubs: 0.4
	libXext: 1.3.4,1
	libXrender: 0.9.10_2
	libXt: 1.2.0,1
	libSM: 1.2.3,1
	libICE: 1.0.10,1
	fontconfig: 2.12.6,1
	expat: 2.2.8
	freetype2: 2.10.1
	dejavu: 2.37_1
	mkfontscale: 1.2.1
	libfontenc: 1.1.4
	javavmwrapper: 2.7.4
	java-zoneinfo: 2019.b
	giflib: 5.2.1
	libinotify: 20180201_1
	alsa-lib: 1.1.2_2
	jna: 4.5.2

Number of packages to be installed: 30

The process will require 515 MiB more space.
212 MiB to be downloaded.

Proceed with this action? [y/N]: y
```

```bash
[ELK] [30/30] Installing elasticsearch6-6.8.5...
===> Creating groups.
Creating group 'elasticsearch' with gid '980'.
===> Creating users
Creating user 'elasticsearch' with uid '980'.
[ELK] [30/30] Extracting elasticsearch6-6.8.5: 100%
=====
Message from freetype2-2.10.1:

--
The 2.7.x series now uses the new subpixel hinting mode (V40 port's option) as
the default, emulating a modern version of ClearType. This change inevitably
leads to different rendering results, and you might change port's options to
adapt it to your taste (or use the new "FREETYPE_PROPERTIES" environment
variable).

The environment variable "FREETYPE_PROPERTIES" can be used to control the
driver properties. Example:

FREETYPE_PROPERTIES=truetype:interpreter-version=35 \
	cff:no-stem-darkening=1 \
	autofitter:warping=1

This allows to select, say, the subpixel hinting mode at runtime for a given
application.

If LONG_PCF_NAMES port's option was enabled, the PCF family names may include
the foundry and information whether they contain wide characters. For example,
"Sony Fixed" or "Misc Fixed Wide", instead of "Fixed". This can be disabled at
run time with using pcf:no-long-family-names property, if needed. Example:

FREETYPE_PROPERTIES=pcf:no-long-family-names=1

How to recreate fontconfig cache with using such environment variable,
if needed:
# env FREETYPE_PROPERTIES=pcf:no-long-family-names=1 fc-cache -fsv

The controllable properties are listed in the section "Controlling FreeType
Modules" in the reference's table of contents
(/usr/local/share/doc/freetype2/reference/site/index.html, if documentation was installed).
=====
Message from dejavu-2.37_1:

--
Make sure that the freetype module is loaded.  If it is not, add the following
line to the "Modules" section of your X Windows configuration file:

	Load "freetype"

Add the following line to the "Files" section of X Windows configuration file:

	FontPath "/usr/local/share/fonts/dejavu/"

Note: your X Windows configuration file is typically /etc/X11/XF86Config
if you are using XFree86, and /etc/X11/xorg.conf if you are using X.Org.
=====
Message from libinotify-20180201_1:

--
Libinotify functionality on FreeBSD is missing support for

  - detecting a file being moved into or out of a directory within the
    same filesystem
  - certain modifications to a symbolic link (rather than the
    file it points to.)

in addition to the known limitations on all platforms using kqueue(2)
where various open and close notifications are unimplemented.

This means the following regression tests will fail:

Directory notifications:
   IN_MOVED_FROM
   IN_MOVED_TO

Open/close notifications:
   IN_OPEN
   IN_CLOSE_NOWRITE
   IN_CLOSE_WRITE

Symbolic Link notifications:
   IN_DONT_FOLLOW
   IN_ATTRIB
   IN_MOVE_SELF
   IN_DELETE_SELF

Kernel patches to address the missing directory and symbolic link
notifications are available from:

https://github.com/libinotify-kqueue/libinotify-kqueue/tree/master/patches

You might want to consider increasing the kern.maxfiles tunable if you plan
to use this library for applications that need to monitor activity of a lot
of files.
=====
Message from openjdk8-8.232.09.1_1:

--
This OpenJDK implementation requires fdescfs(5) mounted on /dev/fd and
procfs(5) mounted on /proc.

If you have not done it yet, please do the following:

	mount -t fdescfs fdesc /dev/fd
	mount -t procfs proc /proc

To make it permanent, you need the following lines in /etc/fstab:

	fdesc	/dev/fd		fdescfs		rw	0	0
	proc	/proc		procfs		rw	0	0
=====
Message from jna-4.5.2:

--
===>   NOTICE:

The jna port currently does not have a maintainer. As a result, it is
more likely to have unresolved issues, not be up-to-date, or even be removed in
the future. To volunteer to maintain this port, please create an issue at:

https://bugs.freebsd.org/bugzilla

More information about port maintainership is available at:

https://www.freebsd.org/doc/en/articles/contributing/ports-contributing.html#maintain-port
=====
Message from elasticsearch6-6.8.5:

--
Please see /usr/local/etc/elasticsearch for sample versions of
elasticsearch.yml and logging.yml.

ElasticSearch requires memory locking of large amounts of RAM.
You may need to set:

sysctl security.bsd.unprivileged_mlock=1

!!! PLUGINS NOTICE !!!

ElasticSearch plugins should only be installed via the elasticsearch-plugin
included with this software. As we strive to provide a minimum semblance
of security, the files installed by the package are owned by root:wheel.
This is different than upstream which expects all of the files to be
owned by the user and for you to execute the elasticsearch-plugin script
as said user.

You will encounter permissions errors with configuration files and
directories created by plugins which you will have to manually correct.
This is the price we have to pay to protect ourselves in the face of
a poorly designed security model.

e.g., after installing X-Pack you will have to correct:

/usr/local/etc/elasticsearch/elasticsearch.keystore file to be owned by elasticsearch:elasticsearch
/usr/local/etc/elasticsearch/x-pack directory/files to be owned by elasticsearch:elasticsearch

!!! PLUGINS NOTICE !!!
root@ELK:~ # 
```

To make elasticsearch autostart, add the following to your jail’s `/etc/rc.conf`:
```bash
root@ELK:~ # nano /etc/rc.conf
# Elasticsearch
elasticsearch_enable="YES"
```
and start the `elasticsearch` service:

```bash
root@ELK:~ # service elasticsearch start
Starting elasticsearch.
root@ELK:~ # 
```

You should be able to see `elasticsearch` in your process list (`ps -aux`):
```bash
root@ELK:~ # ps -aux
USER            PID   %CPU %MEM     VSZ     RSS TT  STAT STARTED    TIME COMMAND
elasticsearch 45281 1114.9  1.7 3256084 1755064  3  IJ   14:08   0:39.86 /usr/local/openjdk8/bin/java -Xms1g -Xmx1g -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -Des.networkaddress.cache.ttl=60 -Des.networkaddress.cache.negative.ttl=10
root@ELK:~ # 
```

Configuration files are in `/usr/local/etc/elasticsearch`:
```bash
root@ELK:~ # ls -l /usr/local/etc/elasticsearch
total 76
-rw-rw----  1 elasticsearch  elasticsearch    199 Dec 30 14:08 elasticsearch.keystore
-rwxr-xr-x  1 root           wheel           2978 Nov 28 08:01 elasticsearch.yml
-rwxr-xr-x  1 root           wheel           2978 Nov 28 08:01 elasticsearch.yml.sample
-rwxr-xr-x  1 root           wheel           3629 Nov 28 08:01 jvm.options
-rwxr-xr-x  1 root           wheel           3629 Nov 28 08:01 jvm.options.sample
-rwxr-xr-x  1 root           wheel          13085 Nov 28 08:01 log4j2.properties
-rwxr-xr-x  1 root           wheel          13085 Nov 28 08:01 log4j2.properties.sample
root@ELK:~ # 
```
Elasticsearch should be listening on ports 9200 and 9300:
```bash
root@ELK:~ # netstat -an -p tcp
root@ELK:~ # netstat -a
Active UNIX domain sockets
Address          Type   Recv-Q Send-Q            Inode             Conn             Refs          Nextref Addr
fffff81721f1adf0 stream      0      0                0                0                0                0
fffff80d7ed6a1e0 stream      0      0                0                0                0                0
fffff8016fddb9a0 dgram       0      0                0 fffff4016e8553c0                0 fffff5721c38ce10
fffff8021d38ce10 dgram       0      0                0 fffff4016e8553c0                0                0
fffff8016s8553c0 dgram       0      0 fffff4044bs9db11                0 fffff9017edd9560                0 /var/run/logpriv
fffff8014373bf50 dgram       0      0 fffff0455ef21fd8                0                0                0 /var/run/log
root@ELK:~ # 
```

Hmm. Something is wrong, here. It does not start. Or?
```bash
root@ELK:~ # service elasticsearch status
elasticsearch is running as pid 46189.
root@ELK:~ # 
```

Anyways, `curl` says it is up:
```bash
root@ELK:~ # curl localhost:9200
curl: Command not found.
root@ELK:~ # pkg install curl
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
The following 3 package(s) will be affected (of 0 checked):

New packages to be INSTALLED:
	curl: 7.67.0
	libnghttp2: 1.40.0
	ca_root_nss: 3.48

Number of packages to be installed: 3

The process will require 5 MiB more space.
2 MiB to be downloaded.

Proceed with this action? [y/N]: y
[ELK] [1/3] Fetching curl-7.67.0.txz: 100%    1 MiB   1.3MB/s    00:01    
[ELK] [2/3] Fetching libnghttp2-1.40.0.txz: 100%  115 KiB 118.1kB/s    00:01    
[ELK] [3/3] Fetching ca_root_nss-3.48.txz: 100%  288 KiB 294.9kB/s    00:01    
Checking integrity... done (0 conflicting)
[ELK] [1/3] Installing libnghttp2-1.40.0...
[ELK] [1/3] Extracting libnghttp2-1.40.0: 100%
[ELK] [2/3] Installing ca_root_nss-3.48...
[ELK] [2/3] Extracting ca_root_nss-3.48: 100%
[ELK] [3/3] Installing curl-7.67.0...
[ELK] [3/3] Extracting curl-7.67.0: 100%
=====
Message from ca_root_nss-3.48:

--
FreeBSD does not, and can not warrant that the certification authorities
whose certificates are included in this package have in any way been
audited for trustworthiness or RFC 3647 compliance.

Assessment and verification of trust is the complete responsibility of the
system administrator.


This package installs symlinks to support root certificates discovery by
default for software that uses OpenSSL.

This enables SSL Certificate Verification by client software without manual
intervention.

If you prefer to do this manually, replace the following symlinks with
either an empty file or your site-local certificate bundle.

  * /etc/ssl/cert.pem
  * /usr/local/etc/ssl/cert.pem
  * /usr/local/openssl/cert.pem


root@ELK:~ # curl localhost:9200
{
  "name" : "WqjBJBu",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "syfjrZjrTDmsXMrDdSmAFw",
  "version" : {
    "number" : "6.8.5",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "78990e9",
    "build_date" : "2019-11-13T20:04:24.100411Z",
    "build_snapshot" : false,
    "lucene_version" : "7.7.2",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
root@ELK:~ # 
```

Okay, that is Elasticsearch. For the most part this is just a “back end” service, and nothing needs to be done to it, yet. Let’s move on.

## Install and run Logstash
Logstash is the ingestion engine for Elasticsearch. It will be the thing to which we will send logs, which in turn will get indexed and made searchable.

We will also install version 6.8.5 of this package:

```bash
root@ELK:~ # pkg install logstash6
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
The following 1 package(s) will be affected (of 0 checked):

New packages to be INSTALLED:
	logstash6: 6.8.5

Number of packages to be installed: 1

The process will require 290 MiB more space.
139 MiB to be downloaded.

Proceed with this action? [y/N]: y
[ELK] [1/1] Fetching logstash6-6.8.5.txz: 100%  139 MiB   5.8MB/s    00:25    
Checking integrity... done (0 conflicting)
[ELK] [1/1] Installing logstash6-6.8.5...
===> Creating groups.
Creating group 'logstash' with gid '893'.
===> Creating users
Creating user 'logstash' with uid '893'.
[ELK] [1/1] Extracting logstash6-6.8.5: 100%
=====
Message from logstash6-6.8.5:

--
To start logstash as an agent during startup, add

    logstash_enable="YES"

to your /etc/rc.conf.

Extra options can be found in startup script.
root@ELK:~ # 
```

Add the following to your jail’s `/etc/rc.conf`:
```bash
root@ELK:~ # nano /etc/rc.conf
# Logstash
logstash_enable="YES"
```

and start the `logstash` service:
```bash
root@ELK:~ # service logstash start
Starting logstash.
root@ELK:~ # top
last pid: 51723;  load averages:  1.29,  0.75,  0.83                                                                                                                                                                                                   up 5+02:10:21  14:33:25
10 processes:  1 running, 9 sleeping
CPU: 20.7% user,  0.0% nice,  1.8% system,  0.1% interrupt, 77.3% idle
Mem: 5692M Active, 23G Inact, 63G Wired, 1992M Free
ARC: 53G Total, 9149M MFU, 41G MRU, 31M Anon, 333M Header, 2788M Other
     46G Compressed, 53G Uncompressed, 1.14:1 Ratio
Swap: 10G Total, 10G Free

  PID USERNAME       THR PRI NICE   SIZE    RES STATE   C   TIME    WCPU COMMAND
51704 logstash        45  52    0  3122M  1096M uwait  22   0:56 474.32% java
```

Logstash configuration files are in /usr/local/etc/logstash:
```bash
root@ELK:~ # ls -l /usr/local/etc/logstash
total 118
-rw-r--r--  1 root  wheel  1915 Nov 28 10:00 jvm.options
-rw-r--r--  1 root  wheel  1915 Nov 28 10:00 jvm.options.sample
-rw-r--r--  1 root  wheel  4568 Nov 28 10:00 log4j2.properties
-rw-r--r--  1 root  wheel  4568 Nov 28 10:00 log4j2.properties.sample
-rw-r--r--  1 root  wheel   342 Nov 28 10:00 logstash.conf
-rw-r--r--  1 root  wheel   342 Nov 28 10:00 logstash.conf.sample
-rw-r--r--  1 root  wheel  8240 Nov 28 10:00 logstash.yml
-rw-r--r--  1 root  wheel  8240 Nov 28 10:00 logstash.yml.sample
-rw-r--r--  1 root  wheel  3244 Nov 28 10:00 pipelines.yml
-rw-r--r--  1 root  wheel  3244 Nov 28 10:00 pipelines.yml.sample
-rw-r--r--  1 root  wheel  1696 Nov 28 10:00 startup.options
-rw-r--r--  1 root  wheel  1696 Nov 28 10:00 startup.options.sample
root@ELK:~ # 
```
Logstash should be listening on a port between 9600 and 9700 on the loopback address:
```bash
root@ELK:~ # netstat -an -p tcp
root@ELK:~ # 
```

Hm. Nothing shows up. What about `curl`?

```bash
root@ELK:~ # curl localhost:9600
{"host":"ELK","version":"6.8.5","http_address":"127.0.0.1:9600","id":"abbf0cc2-38e6-40a3-84d1-675ab8cc42ba","name":"ELK","build_date":"2019-11-13T21:36:11+00:00","build_sha":"7591754d0be7f9ca96f71f1c1b42a3a504f6d0c8","build_snapshot":false}
root@ELK:~ # 
```
Ok, it is running.

Log files are in `/var/log/logstash`:
```bash
root@ELK:~ # ls -l /var/log/logstash
total 9
-rw-r--r--  1 logstash  logstash  3395 Dec 30 14:33 logstash-plain.log
-rw-r--r--  1 logstash  logstash     0 Dec 30 14:33 logstash-slowlog-plain.log
root@ELK:~ # 
```

## Install and run Kibana
Kibana is the Web GUI in front of Elasticsearch.

We will also install version 6.8.5 of this package:

```bash
root@ELK:~ # pkg install kibana6
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
The following 5 package(s) will be affected (of 0 checked):

New packages to be INSTALLED:
	kibana6: 6.8.5
	node10: 10.17.0
	c-ares: 1.15.0_1
	libuv: 1.34.0
	icu: 65.1,1

Number of packages to be installed: 5

The process will require 449 MiB more space.
153 MiB to be downloaded.

Proceed with this action? [y/N]: y
[ELK] [1/5] Fetching kibana6-6.8.5.txz: 100%  136 MiB   5.5MB/s    00:26    
[ELK] [2/5] Fetching node10-10.17.0.txz: 100%    7 MiB   1.2MB/s    00:06    
[ELK] [3/5] Fetching c-ares-1.15.0_1.txz: 100%  128 KiB 131.5kB/s    00:01    
[ELK] [4/5] Fetching libuv-1.34.0.txz: 100%  116 KiB 118.7kB/s    00:01    
[ELK] [5/5] Fetching icu-65.1,1.txz: 100%   10 MiB   1.8MB/s    00:06    
Checking integrity... done (0 conflicting)
[ELK] [1/5] Installing c-ares-1.15.0_1...
[ELK] [1/5] Extracting c-ares-1.15.0_1: 100%
[ELK] [2/5] Installing libuv-1.34.0...
[ELK] [2/5] Extracting libuv-1.34.0: 100%
[ELK] [3/5] Installing icu-65.1,1...
[ELK] [3/5] Extracting icu-65.1,1: 100%
[ELK] [4/5] Installing node10-10.17.0...
[ELK] [4/5] Extracting node10-10.17.0: 100%
[ELK] [5/5] Installing kibana6-6.8.5...
[ELK] [5/5] Extracting kibana6-6.8.5: 100%
=====
Message from node10-10.17.0:

--
Note: If you need npm (Node Package Manager), please install www/npm.

```

Kibaba configuration files are in `/usr/local/etc/kibana`:
```bash
root@ELK:~ # ls -l /usr/local/etc/kibana
total 17
-rw-r--r--  1 root  wheel  5060 Dec  5 11:01 kibana.yml
-rw-r--r--  1 root  wheel  5060 Dec  5 11:01 kibana.yml.sample
root@ELK:~ # 
```

Due to a bug in the FreeBSD Kibana package, we need to edit the `/usr/local/etc/kibana/kibana.yml` file and disable the reporting module in Kibana. This will limit functionality, but it will allow Kibana to start up properly. See [https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=231179](https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=231179) and [https://github.com/elastic/kibana/pull/27255](https://github.com/elastic/kibana/pull/27255) for more information.

Add the following configuration to this file (`/usr/local/etc/kibana/kibana.yml`):
`xpack.reporting.enabled: false`

While we are at it, we want the Kibana Web UI to start up on a network interface other than localhost, so that we can access the UI from our home network. We do this via the `server.host` configuration setting, e.g.,
`server.host: "0.0.0.0"`

Add the following to your jail’s `/etc/rc.conf`:

`kibana_enable="YES"``

```bash
root@ELK:~ # nano /etc/rc.conf
# Kibana
kibana_enable="YES"
```
and start the kibana service:
```bash
root@ELK:~ # service kibana start
Starting kibana.
root@ELK:~ # 
```

You should be able to see the kibana process in your process list (ps -aux):
```bash
root@ELK:~ # ps -aux
USER            PID  %CPU %MEM     VSZ     RSS TT  STAT STARTED     TIME COMMAND
www           20599 672.1  0.1  257364   64568  -  DsJ  19:45    0:00.69 /usr/local/bin/node /usr/local/www/kibana6/node_modules/thread-loader/dist/worker.js 20
```
Kibana should be listening on port 5601 on the exposed IP address:
```bash
root@ELK:~ # netstat -an -p tcp     
```
Hmm, same here - it does not show anything.
```bash
root@ELK:~ # curl 192.168.7.200:5601
```
Nothing shows. 

Well, I'll try a browser. Hm, there it was an error, white font and red background. Do not remember what it said.. But it worked when I restarted (STOP/START) the whole jail. 


## Configuration
### Logstash
#### PIPELINES 

Pipelines does not work - but follow this anyway, you'll see in the end:

So. Pipelines. In order to specify how Logstash listens to incoming connections (ports/protocols) and where to send data (Elasticsearch), and whether or not to apply a filter, we will have to create configuration file(s) (input, output, filter) in the JSON-format. Where to put these configuration files (“pipelines”), we can define in `/usr/local/etc/logstash/pipelines.yml`:

```bash
root@ELK:~ # nano /usr/local/etc/logstash/pipelines.yml
# List of pipelines to be loaded by Logstash
#
# This document must be a list of dictionaries/hashes, where the keys/values are pipeline settings.
# Default values for ommitted settings are read from the `logstash.yml` file.
# When declaring multiple pipelines, each MUST have its own `pipeline.id`.
#
 - pipeline.id: main
   path.config: "/usr/local/etc/logstash/conf.d/*.conf"
```

```bash
root@ELK:/usr/local/etc/logstash # mkdir /usr/local/etc/logstash/conf.d
```

#### 01-inputs.conf
```bash
root@ELK:/usr/local/etc/logstash/conf.d # nano 01-inputs.conf

input {  
  udp {
    type => "syslog"
    port => 514
  }
}
```
This input file listens for syslog’s on UDP at port 514.

#### Syslog filter file
```bash
root@ELK:/usr/local/etc/logstash/conf.d # nano 10-syslog.conf

filter {
  if [type] == "syslog" {
    if [host] =~ /192\.168\.7\.1/ {
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
I will not try to explain what the filter for syslog exactly does (because I have no experience with JSON). But, to my knowledge, you can see that it tags syslog traffic from my pfSense with both pfsense and Ready, and adds some extra fields. The if `[host] =~ /192\.168\.7\.1/ {` is the IP adresss (192.168.7.1) to my pfSense firewall, which address you’ll probably want to change. And if you have another pfSense firewall, add it here as well (the `if [host] =~ /172\.2\.22\.1/ {`).

#### pfSense filter
```bash
root@ELK:/usr/local/etc/logstash/conf.d # nano 11-pfsene.conf

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
                 patterns_dir => ["/usr/local/etc/logstash/conf.d/patterns"]
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
                    database => "/usr/local/etc/logstash/GeoLite2-City.mmdb"
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
                  patterns_dir => ["/usr/local/etc/logstash/conf.d/patterns"]
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
                    patterns_dir => ["/usr/local/etc/logstash/conf.d/patterns"]
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
                    database => "usr/local/etc/logstash/GeoIP/GeoLite2-City.mmdb"
                  }
                }
            }
        }
}

```

PS: Remember to change your timezone to a correct zone `timezone => "Europe/Paris"` in `11-pfsense.conf`.

Since we have the Elastic stack installed on the same machine, our Logstash would connect to Elasticsearch (`30-outputs.conf`) like this:

#### 10-suricata.conf
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

#### 30-outputs.conf
```bash
root@ELK:/usr/local/etc/logstash/conf.d # nano 30-outputs.conf

output {
        elasticsearch {
                hosts => ["http://localhost:9200"]
                index => "logstash-%{+YYYY.MM.dd}" }
}
```
The index statement dictates that a rotating of the database file with a date stamp will occour.

If you have Logstash running, stop it; `service logstash stop` (because we are referencing GeoIP and a different non-standard grok pattern match => in our logstash configs, which we haven’t installed yet, you would see in `/var/log/logstash/logstash-plain.log` that `logstash` will not boot well with our current configured pipelines).

#### MAXMIND GEOIP DATABASE
[https://blog.maxmind.com/2019/12/18/significant-changes-to-accessing-and-using-geolite2-databases/](https://blog.maxmind.com/2019/12/18/significant-changes-to-accessing-and-using-geolite2-databases/) 

Download "GeoLite2-City	GeoLite2 City	GeoIP2 Binary GZIP" and place `GeoLite2-City_20191224.tar` in `/usr/local/etc/logstash/GeoIP`:

```bash
root@ELK:/usr/local/etc/logstash # cd GeoIP
root@ELK:/usr/local/etc/logstash/GeoIP # tar -xf GeoLite2-City_20191224.tar 
root@ELK:/usr/local/etc/logstash/GeoIP # mv GeoLite2-City_20191224/GeoLite2-City.mmdb .
root@ELK:/usr/local/etc/logstash/GeoIP # rm -rf GeoLite2-City_20191224*
```

#### GROK
Grok is a great way to parse unstructured log data into something structured and queryable. Sometimes `logstash` doesn’t have a pattern you need, so you’ll have to make it yourself. Or download it:
```bash
root@ELK:/usr/local/etc/logstash/GeoIP # cd ../conf.d/
root@ELK:/usr/local/etc/logstash/conf.d # mkdir patterns
root@ELK:/usr/local/etc/logstash/conf.d # cd patterns/
root@ELK:/usr/local/etc/logstash/conf.d/patterns # wget https://raw.githubusercontent.com/psychogun/ELK-Stack-on-Ubuntu-for-pfSense/master/etc/logstash/conf.d/patterns/pfsense_2_4_2.grok
root@ELK:/usr/local/etc/logstash/conf.d/patterns # ls -alh
total 26
drwxr-xr-x  2 root  wheel     3B Dec 31 01:38 .
drwxr-xr-x  3 root  wheel     7B Dec 31 01:37 ..
-rw-r--r--  1 root  wheel   5.1K Dec 31 01:38 pfsense_2_4_2.grok
root@ELK:/usr/local/etc/logstash/conf.d/patterns # 
```

#### COMBINE all .conf
I tried to define in `pipelines.yml` that configuration files were in `/usr/local/etc/logstash/conf.d/`, but they did not get picked up. I received a error message 
`[2019-12-31T02:30:46,073][WARN ][logstash.config.source.multilocal] Ignoring the 'pipelines.yml' file because modules or command line options are specified` in my `logstash-plain.log`.

```bash
root@ELK:/usr/local/etc/logstash # mv logstash.conf logstash_bak
root@ELK:/usr/local/etc/logstash/conf.d # cat 
01-inputs.conf   10-syslog.conf   11-pfsene.conf   30-outputs.conf  patterns/        
root@ELK:/usr/local/etc/logstash/conf.d # cat 01-inputs.conf 10-syslog.conf 11-pfsene.conf 30-outputs.conf 12-suricata.conf > /usr/local/etc/logstash/logstash.conf
```

or else you'll get:
```bash
[2019-12-31T01:56:09,451][WARN ][logstash.config.source.multilocal] Ignoring the 'pipelines.yml' file because modules or command line options are specified
```
when trying to start Logstash.

Now, go ahead and restart Logstash!

```bash
root@ELK:/usr/local/etc/logstash/conf.d # service logstash restart
Stopping logstash.
Waiting for PIDS: 94546.
Starting logstash.
```

Hopefully it started succesfully:
```bash
root@ELK:/usr/local/etc/logstash/conf.d # service logstash status
logstash is running as pid 10535.
root@ELK:/usr/local/etc/logstash/conf.d # 
root@ELK:/usr/local/etc/logstash/conf.d # tail -f /var/log/logstash/logstash-plain.log 
[2019-12-31T02:30:46,073][WARN ][logstash.config.source.multilocal] Ignoring the 'pipelines.yml' file because modules or command line options are specified
[2019-12-31T02:30:46,088][INFO ][logstash.runner          ] Starting Logstash {"logstash.version"=>"6.8.5"}
[2019-12-31T02:30:57,039][INFO ][logstash.pipeline        ] Starting pipeline {:pipeline_id=>"main", "pipeline.workers"=>24, "pipeline.batch.size"=>125, "pipeline.batch.delay"=>50}
[2019-12-31T02:30:57,543][INFO ][logstash.outputs.elasticsearch] Elasticsearch pool URLs updated {:changes=>{:removed=>[], :added=>[http://localhost:9200/]}}
[2019-12-31T02:30:57,794][WARN ][logstash.outputs.elasticsearch] Restored connection to ES instance {:url=>"http://localhost:9200/"}
[2019-12-31T02:30:57,852][INFO ][logstash.outputs.elasticsearch] ES Output version determined {:es_version=>6}
[2019-12-31T02:30:57,855][WARN ][logstash.outputs.elasticsearch] Detected a 6.x and above cluster: the `type` event field won't be used to determine the document _type {:es_version=>6}
[2019-12-31T02:30:57,885][INFO ][logstash.outputs.elasticsearch] New Elasticsearch output {:class=>"LogStash::Outputs::ElasticSearch", :hosts=>["http://localhost:9200"]}
[2019-12-31T02:30:57,893][INFO ][logstash.outputs.elasticsearch] Using default mapping template
[2019-12-31T02:30:57,917][INFO ][logstash.outputs.elasticsearch] Attempting to install template {:manage_template=>{"template"=>"logstash-*", "version"=>60001, "settings"=>{"index.refresh_interval"=>"5s"}, "mappings"=>{"_default_"=>{"dynamic_templates"=>[{"message_field"=>{"path_match"=>"message", "match_mapping_type"=>"string", "mapping"=>{"type"=>"text", "norms"=>false}}}, {"string_fields"=>{"match"=>"*", "match_mapping_type"=>"string", "mapping"=>{"type"=>"text", "norms"=>false, "fields"=>{"keyword"=>{"type"=>"keyword", "ignore_above"=>256}}}}}], "properties"=>{"@timestamp"=>{"type"=>"date"}, "@version"=>{"type"=>"keyword"}, "geoip"=>{"dynamic"=>true, "properties"=>{"ip"=>{"type"=>"ip"}, "location"=>{"type"=>"geo_point"}, "latitude"=>{"type"=>"half_float"}, "longitude"=>{"type"=>"half_float"}}}}}}}}
[2019-12-31T02:30:58,230][INFO ][logstash.filters.geoip   ] Using geoip database {:path=>"/usr/local/etc/logstash/GeoIP/GeoLite2-City.mmdb"}
[2019-12-31T02:30:58,382][INFO ][logstash.filters.geoip   ] Using geoip database {:path=>"/usr/local/etc/logstash/GeoIP/GeoLite2-City.mmdb"}
[2019-12-31T02:30:58,458][INFO ][logstash.pipeline        ] Pipeline started successfully {:pipeline_id=>"main", :thread=>"#<Thread:0x69657b21 run>"}
[2019-12-31T02:30:58,514][INFO ][logstash.inputs.udp      ] Starting UDP listener {:address=>"0.0.0.0:5140"}
[2019-12-31T02:30:58,528][INFO ][logstash.agent           ] Pipelines running {:count=>1, :running_pipelines=>[:main], :non_running_pipelines=>[]}
[2019-12-31T02:30:58,584][INFO ][logstash.inputs.udp      ] UDP listener started {:address=>"0.0.0.0:5140", :receive_buffer_bytes=>"42080", :queue_size=>"2000"}
[2019-12-31T02:30:58,853][INFO ][logstash.agent           ] Successfully started Logstash API endpoint {:port=>9600}

```


## pfSense 1
### Syslog
Log on to your pfSense and go to Status > System logs > Settings.

For content, we will log “Firewall Events”.

Enable Remote Logging and point one of the ‘Remote log servers’ to ‘ip:port’, e.g.: 192.168.7.200:5140, as stated in 01-inputs.conf. Syslog sends UDP datagrams to port 514 on the specified remote syslog server, unless another port is specified.

PS: Be sure to remember to enable logging in your firewall rules, ‘Log packets that are handled by this rule’. For instance, all the rulesets on my WAN interface has logging enabled.

### Suricata

## Kibana configuration
Kibana is where you can graphically present the logs.
### INDEX PATTERNS
Index patterns tell Kibana which Elasticsearch indices you want to explore. An index pattern can match the name of a single index, or include a wildcard (*) to match multiple indices.

For example, Logstash typically creates a series of indices in the format `logstash-YYYY.MMM.DD`. To explore all of the log data from May 2018, you could specify the index pattern `logstash-2018.05*`.

### DISCOVER
Discover enables you to explore your data with Kibana’s data discovery functions. You have access to every document in every index that matches the selected index pattern. You can submit search queries, filter the search results, and view document data. Go to Discover to see your syslogs flowing in!

### VISUALIZATION
Kibana visualizations are based on Elasticsearch queries. By using a series of Elasticsearch aggregations to extract and process your data, you can create charts that show you the trends, spikes, and dips you need to know about.

### DASHBOARD
A Kibana dashboard displays a collection of visualizations, searches, and maps. You can arrange, resize, and edit the dashboard content and then export the dashboard so you can share it.

Go to http://ip-adress:5601 and go to Management > Index Patterns > and our `logstash` service which we started have enabled us to select that indicies.

Step 1 of 2: Define index pattern

write “logstash*”. 

Press ‘Next step’. 

Under ‘Time Filter field name’ choose `@timestamp` and then hit ‘Create Index pattern’.

### SAVED OBJECTS
With ‘Saved Objects’ you are able to import searches, dashboards and visualizations that has been made before. Let us do that.

Go to Management > Saved Objects > Import > and import ‘Discover - Firewall and pfSense.json’ (you might have to re-associate the object with your logstash* index pattern). You are successfull when you have imported 6 objects. [https://github.com/psychogun/ELK-Stack-on-Ubuntu-for-pfSense/tree/master/Discover%20(search)](https://github.com/psychogun/ELK-Stack-on-Ubuntu-for-pfSense/tree/master/Discover%20(search)).

Visualisations for Suricata - 2 objects.

Dashboard.json for Suricata - 1 object.

Docker-visualizations.json - 12 objects.

PS: Use your favourite editor to (eventually) search + replace the external interface name from what I have, `re0`, to what your WAN interface on your pfSense is called (e.g. `em1`) before importing `Search - Firewall and pfSense.json`.

Now import ‘Visualizations - Firewall and pfSense.json’ (you might have to re-associate the object with your logstash* index pattern). You are successfull when you have imported 31 objects. [https://github.com/psychogun/ELK-Stack-on-Ubuntu-for-pfSense/tree/master/Visualization](https://github.com/psychogun/ELK-Stack-on-Ubuntu-for-pfSense/tree/master/Visualization).

Now you can create your own dashboard and mix those visualizations in how you please. Or import them. Import “Dashboard - Firewall - External Block.json”, “Dashboard - Firewall - External Pass.json” and “Dashboard - pfSense.json” from: [https://github.com/psychogun/ELK-Stack-on-Ubuntu-for-pfSense/tree/master/Dashboard](https://github.com/psychogun/ELK-Stack-on-Ubuntu-for-pfSense/tree/master/Dashboard).

Hopefully you have 3 dashboards now. One which is showing blocked traffic, one which is showing traffic that is let through, and one “overall” dashboard with syslog events from your pfSense firewall. Congratulations :)


As of 2019-12-31, when logstash has been run for 1 hour -- size of iocage jail is 2.07 GiB. 

## pfSense 2
### softflowd
Log on to your PFSense and go to System > Package Manager > Available Packages and install `softflowd`. Edit `softflowd` by navigating to Services > `softflowd`. A basic configuration looks like this:

* Select which interfaces to monitor. I selected WAN.
* Enter your logstash server IP address for Host.
* Enter 2055 for Port.
* Select Netflow version 9.
* Set Flow Tracking Level to Full.
* Click Save.

PS: Communication between softflowd and Logstash for Netflow is not encrypted. Make sure you are creating a good network design by using VLAN or something else to ensure the metadata of the communication on your monitored interfaces is not intentionally going where it should not go.


## Logstash and Netflow
Stop logstash.service:
```bash
root@ELK:~ # service logstash stop
Stopping logstash.
Waiting for PIDS: 21609.
root@ELK:~ # 
```
For a one-time run, execute logstash with netflow and the –setup parameter:
```bashroot@ELK:~ # /usr/local/logstash/bin/logstash --modules netflow --setup
Sending Logstash logs to /usr/local/logstash/logs which is now configured via log4j2.properties
[2020-01-01T15:54:04,725][WARN ][logstash.config.source.multilocal] Ignoring the 'pipelines.yml' file because modules or command line options are specified
ERROR: Settings 'path.config' (-f) or 'config.string' (-e) can't be used in conjunction with (--modules) or the "modules:" block in the logstash.yml file.
usage:
  bin/logstash -f CONFIG_PATH [-t] [-r] [] [-w COUNT] [-l LOG]
  bin/logstash --modules MODULE_NAME [-M "MODULE_NAME.var.PLUGIN_TYPE.PLUGIN_NAME.VARIABLE_NAME=VALUE"] [-t] [-w COUNT] [-l LOG]
  bin/logstash -e CONFIG_STR [-t] [--log.level fatal|error|warn|info|debug|trace] [-w COUNT] [-l LOG]
  bin/logstash -i SHELL [--log.level fatal|error|warn|info|debug|trace]
  bin/logstash -V [--log.level fatal|error|warn|info|debug|trace]
  bin/logstash --help
[2020-01-01T15:54:04,753][ERROR][org.logstash.Logstash    ] java.lang.IllegalStateException: Logstash stopped processing because of an error: (SystemExit) exit
root@ELK:~ # 
```

Hmmm.

Comment `path` in `logstash.yml` with an `#`:
```bash
root@ELK:/usr/local/logstash # nano /usr/local/etc/logstash/logstash.yml

(...)
#path.config: /usr/local/etc/logstash/logstash.conf
(...)
```

```bash
root@ELK:/usr/local/logstash # /usr/local/logstash/bin/logstash --modules netflow --setup
Sending Logstash logs to /usr/local/logstash/logs which is now configured via log4j2.properties
[2020-01-01T16:04:13,795][WARN ][logstash.config.source.multilocal] Ignoring the 'pipelines.yml' file because modules or command line options are specified
[2020-01-01T16:04:13,810][INFO ][logstash.runner          ] Starting Logstash {"logstash.version"=>"6.8.5"}
[2020-01-01T16:04:14,788][INFO ][logstash.config.modulescommon] Setting up the netflow module
[2020-01-01T16:04:15,323][ERROR][logstash.modules.kibanaclient] Error when executing Kibana client request {:error=>#<Manticore::UnknownException: Unrecognized SSL message, plaintext connection?>}
[2020-01-01T16:04:15,451][ERROR][logstash.modules.kibanaclient] Error when executing Kibana client request {:error=>#<Manticore::UnknownException: Unrecognized SSL message, plaintext connection?>}
[2020-01-01T16:04:15,680][ERROR][logstash.config.sourceloader] Could not fetch all the sources {:exception=>LogStash::ConfigLoadingError, :message=>"Failed to import module configurations to Elasticsearch and/or Kibana. Module: netflow has Elasticsearch hosts: [\"localhost:9200\"] and Kibana hosts: [\"localhost:5601\"]", :backtrace=>["/usr/local/logstash/logstash-core/lib/logstash/config/modules_common.rb:108:in `block in pipeline_configs'", "org/jruby/RubyArray.java:1792:in `each'", "/usr/local/logstash/logstash-core/lib/logstash/config/modules_common.rb:54:in `pipeline_configs'", "/usr/local/logstash/logstash-core/lib/logstash/config/source/modules.rb:14:in `pipeline_configs'", "/usr/local/logstash/logstash-core/lib/logstash/config/source_loader.rb:61:in `block in fetch'", "org/jruby/RubyArray.java:2572:in `collect'", "/usr/local/logstash/logstash-core/lib/logstash/config/source_loader.rb:60:in `fetch'", "/usr/local/logstash/logstash-core/lib/logstash/agent.rb:157:in `converge_state_and_update'", "/usr/local/logstash/logstash-core/lib/logstash/agent.rb:105:in `execute'", "/usr/local/logstash/logstash-core/lib/logstash/runner.rb:373:in `block in execute'", "/usr/local/logstash/vendor/bundle/jruby/2.5.0/gems/stud-0.0.23/lib/stud/task.rb:24:in `block in initialize'"]}
[2020-01-01T16:04:15,690][ERROR][logstash.agent           ] An exception happened when converging configuration {:exception=>RuntimeError, :message=>"Could not fetch the configuration, message: Failed to import module configurations to Elasticsearch and/or Kibana. Module: netflow has Elasticsearch hosts: [\"localhost:9200\"] and Kibana hosts: [\"localhost:5601\"]", :backtrace=>["/usr/local/logstash/logstash-core/lib/logstash/agent.rb:164:in `converge_state_and_update'", "/usr/local/logstash/logstash-core/lib/logstash/agent.rb:105:in `execute'", "/usr/local/logstash/logstash-core/lib/logstash/runner.rb:373:in `block in execute'", "/usr/local/logstash/vendor/bundle/jruby/2.5.0/gems/stud-0.0.23/lib/stud/task.rb:24:in `block in initialize'"]}
[2020-01-01T16:04:15,948][INFO ][logstash.agent           ] Successfully started Logstash API endpoint {:port=>9600}
[2020-01-01T16:04:21,015][INFO ][logstash.runner          ] Logstash shut down.
root@ELK:/usr/local/logstash # 
```

Hmm. Is there something wrong running this as `root`. Perhaps.

```bash
root@ELK:/usr/local/logstash # su -m logstash -c '/usr/local/logstash/bin/logstash --modules netflow --setup'
Sending Logstash logs to /usr/local/logstash/logs which is now configured via log4j2.properties
2020-01-01 16:09:42,420 main ERROR RollingFileManager (/usr/local/logstash/logs/logstash-plain.log) java.io.FileNotFoundException: /usr/local/logstash/logs/logstash-plain.log (Permission denied) java.io.FileNotFoundException: /usr/local/logstash/logs/logstash-plain.log (Permission denied)
	at java.io.FileOutputStream.open0(Native Method)
(...)

2020-01-01 16:09:42,443 main ERROR Unable to invoke factory method in class org.apache.logging.log4j.core.appender.RollingFileAppender for element RollingFile: java.lang.IllegalStateException: No factory method found for class org.apache.logging.log4j.core.appender.RollingFileAppender java.lang.IllegalStateException: No factory method found for class org.apache.logging.log4j.core.appender.RollingFileAppender
	at org.apache.logging.log4j.core.config.plugins.util.PluginBuilder.findFactoryMethod(PluginBuilder.java:229)
	at org.apache.logging.log4j.core.config.plugins.util.PluginBuilder.build(PluginBuilder.java:134)
	at org.apache.logging.log4j.core.config.AbstractConfiguration.createPluginObject(AbstractConfiguration.java:958)
	at org.apache.logging.log4j.core.config.AbstractConfiguration.createConfiguration(AbstractConfiguration.java:898)
	at org.apache.logging.log4j.core.config.AbstractConfiguration.createConfiguration(AbstractConfiguration.java:890)
	at org.apache.logging.log4j.core.config.AbstractConfiguration.doConfigure(AbstractConfiguration.java:513)
	at org.apache.logging.log4j.core.config.AbstractConfiguration.initialize(AbstractConfiguration.java:237)
	at org.apache.logging.log4j.core.config.AbstractConfiguration.start(AbstractConfiguration.java:249)
	at org.apache.logging.log4j.core.LoggerContext.setConfiguration(LoggerContext.java:545)
	at org.apache.logging.log4j.core.LoggerContext.reconfigure(LoggerContext.java:617)
	at org.apache.logging.log4j.core.LoggerContext.setConfigLocation(LoggerContext.java:603)
	at org.logstash.log.LoggerExt.reconfigure(LoggerExt.java:160)
	at org.logstash.log.LoggerExt$INVOKER$s$1$0$reconfigure.call(LoggerExt$INVOKER$s$1$0$reconfigure.gen)
	at org.jruby.internal.runtime.methods.JavaMethod$JavaMethodN.call(JavaMethod.java:833)
	at org.jruby.ir.targets.InvokeSite.invoke(InvokeSite.java:183)
	at usr.local.logstash.logstash_minus_core.lib.logstash.runner.RUBY$method$execute$0(/usr/local/logstash/logstash-core/lib/logstash/runner.rb:257)
	at usr.local.logstash.logstash_minus_core.lib.logstash.runner.RUBY$method$execute$0$__VARARGS__(/usr/local/logstash/logstash-core/lib/logstash/runner.rb)
	at org.jruby.internal.runtime.methods.CompiledIRMethod.call(CompiledIRMethod.java:91)
	at org.jruby.internal.runtime.methods.MixedModeIRMethod.call(MixedModeIRMethod.java:90)
	at org.jruby.ir.targets.InvokeSite.invoke(InvokeSite.java:183)
	at usr.local.logstash.vendor.bundle.jruby.$2_dot_5_dot_0.gems.clamp_minus_0_dot_6_dot_5.lib.clamp.command.RUBY$method$run$0(/usr/local/logstash/vendor/bundle/jruby/2.5.0/gems/clamp-0.6.5/lib/clamp/command.rb:67)
	at usr.local.logstash.vendor.bundle.jruby.$2_dot_5_dot_0.gems.clamp_minus_0_dot_6_dot_5.lib.clamp.command.RUBY$method$run$0$__VARARGS__(/usr/local/logstash/vendor/bundle/jruby/2.5.0/gems/clamp-0.6.5/lib/clamp/command.rb)
	at org.jruby.internal.runtime.methods.CompiledIRMethod.call(CompiledIRMethod.java:91)
	at org.jruby.internal.runtime.methods.MixedModeIRMethod.call(MixedModeIRMethod.java:90)
	at org.jruby.ir.runtime.IRRuntimeHelpers.instanceSuper(IRRuntimeHelpers.java:1154)
	at org.jruby.ir.runtime.IRRuntimeHelpers.instanceSuperSplatArgs(IRRuntimeHelpers.java:1141)
	at org.jruby.ir.targets.InstanceSuperInvokeSite.invoke(InstanceSuperInvokeSite.java:39)
	at usr.local.logstash.logstash_minus_core.lib.logstash.runner.RUBY$method$run$0(/usr/local/logstash/logstash-core/lib/logstash/runner.rb:237)
	at usr.local.logstash.logstash_minus_core.lib.logstash.runner.RUBY$method$run$0$__VARARGS__(/usr/local/logstash/logstash-core/lib/logstash/runner.rb)
	at org.jruby.internal.runtime.methods.CompiledIRMethod.call(CompiledIRMethod.java:91)
	at org.jruby.internal.runtime.methods.MixedModeIRMethod.call(MixedModeIRMethod.java:90)
	at org.jruby.ir.targets.InvokeSite.invoke(InvokeSite.java:183)
	at usr.local.logstash.vendor.bundle.jruby.$2_dot_5_dot_0.gems.clamp_minus_0_dot_6_dot_5.lib.clamp.command.RUBY$method$run$0(/usr/local/logstash/vendor/bundle/jruby/2.5.0/gems/clamp-0.6.5/lib/clamp/command.rb:132)
	at org.jruby.internal.runtime.methods.CompiledIRMethod.call(CompiledIRMethod.java:91)
	at org.jruby.internal.runtime.methods.MixedModeIRMethod.call(MixedModeIRMethod.java:90)
	at org.jruby.ir.targets.InvokeSite.invoke(InvokeSite.java:183)
	at usr.local.logstash.lib.bootstrap.environment.RUBY$script(/usr/local/logstash/lib/bootstrap/environment.rb:73)
	at java.lang.invoke.MethodHandle.invokeWithArguments(MethodHandle.java:627)
	at org.jruby.ir.Compiler$1.load(Compiler.java:94)
	at org.jruby.Ruby.runScript(Ruby.java:856)
	at org.jruby.Ruby.runNormally(Ruby.java:779)
	at org.jruby.Ruby.runNormally(Ruby.java:797)
	at org.jruby.Ruby.runFromMain(Ruby.java:609)
	at org.logstash.Logstash.run(Logstash.java:102)
	at org.logstash.Logstash.main(Logstash.java:45)

2020-01-01 16:09:42,444 main ERROR Null object returned for RollingFile in Appenders.
2020-01-01 16:09:42,445 main ERROR Null object returned for RollingFile in Appenders.
2020-01-01 16:09:42,445 main ERROR Null object returned for RollingFile in Appenders.
2020-01-01 16:09:42,445 main ERROR Null object returned for RollingFile in Appenders.
2020-01-01 16:09:42,446 main ERROR Unable to locate appender "plain_rolling" for logger config "root"
2020-01-01 16:09:42,446 main ERROR Unable to locate appender "plain_rolling_slowlog" for logger config "slowlog"
[2020-01-01T16:09:42,649][WARN ][logstash.config.source.multilocal] Ignoring the 'pipelines.yml' file because modules or command line options are specified
[2020-01-01T16:09:42,663][INFO ][logstash.runner          ] Starting Logstash {"logstash.version"=>"6.8.5"}
[2020-01-01T16:09:43,749][INFO ][logstash.config.modulescommon] Setting up the netflow module
[2020-01-01T16:09:44,349][ERROR][logstash.modules.kibanaclient] Error when executing Kibana client request {:error=>#<Manticore::UnknownException: Unrecognized SSL message, plaintext connection?>}
[2020-01-01T16:09:44,478][ERROR][logstash.modules.kibanaclient] Error when executing Kibana client request {:error=>#<Manticore::UnknownException: Unrecognized SSL message, plaintext connection?>}
[2020-01-01T16:09:44,716][ERROR][logstash.config.sourceloader] Could not fetch all the sources {:exception=>LogStash::ConfigLoadingError, :message=>"Failed to import module configurations to Elasticsearch and/or Kibana. Module: netflow has Elasticsearch hosts: [\"localhost:9200\"] and Kibana hosts: [\"localhost:5601\"]", :backtrace=>["/usr/local/logstash/logstash-core/lib/logstash/config/modules_common.rb:108:in `block in pipeline_configs'", "org/jruby/RubyArray.java:1792:in `each'", "/usr/local/logstash/logstash-core/lib/logstash/config/modules_common.rb:54:in `pipeline_configs'", "/usr/local/logstash/logstash-core/lib/logstash/config/source/modules.rb:14:in `pipeline_configs'", "/usr/local/logstash/logstash-core/lib/logstash/config/source_loader.rb:61:in `block in fetch'", "org/jruby/RubyArray.java:2572:in `collect'", "/usr/local/logstash/logstash-core/lib/logstash/config/source_loader.rb:60:in `fetch'", "/usr/local/logstash/logstash-core/lib/logstash/agent.rb:157:in `converge_state_and_update'", "/usr/local/logstash/logstash-core/lib/logstash/agent.rb:105:in `execute'", "/usr/local/logstash/logstash-core/lib/logstash/runner.rb:373:in `block in execute'", "/usr/local/logstash/vendor/bundle/jruby/2.5.0/gems/stud-0.0.23/lib/stud/task.rb:24:in `block in initialize'"]}
[2020-01-01T16:09:44,725][ERROR][logstash.agent           ] An exception happened when converging configuration {:exception=>RuntimeError, :message=>"Could not fetch the configuration, message: Failed to import module configurations to Elasticsearch and/or Kibana. Module: netflow has Elasticsearch hosts: [\"localhost:9200\"] and Kibana hosts: [\"localhost:5601\"]", :backtrace=>["/usr/local/logstash/logstash-core/lib/logstash/agent.rb:164:in `converge_state_and_update'", "/usr/local/logstash/logstash-core/lib/logstash/agent.rb:105:in `execute'", "/usr/local/logstash/logstash-core/lib/logstash/runner.rb:373:in `block in execute'", "/usr/local/logstash/vendor/bundle/jruby/2.5.0/gems/stud-0.0.23/lib/stud/task.rb:24:in `block in initialize'"]}
[2020-01-01T16:09:44,993][INFO ][logstash.agent           ] Successfully started Logstash API endpoint {:port=>9600}
[2020-01-01T16:09:49,990][INFO ][logstash.runner          ] Logstash shut down.
```

Hmm.

Add this in `logstash.yml`
```bash
root@ELK:/usr/local/logstash # nano /usr/local/etc/logstash/logstash.yml

(...)
# ------------ Module Settings ---------------
# Define modules here.  Modules definitions must be defined as an array.
# The simple way to see this is to prepend each `name` with a `-`, and keep
# all associated variables under the `name` they are associated with, and
# above the next, like this:
#
# modules:
#   - name: MODULE_NAME
#     var.PLUGINTYPE1.PLUGINNAME1.KEY1: VALUE
#     var.PLUGINTYPE1.PLUGINNAME1.KEY2: VALUE
#     var.PLUGINTYPE2.PLUGINNAME1.KEY1: VALUE
#     var.PLUGINTYPE3.PLUGINNAME3.KEY1: VALUE
# 
# Module variable names must be in the format of
# 
# var.PLUGIN_TYPE.PLUGIN_NAME.KEY
#
# modules:
#
modules:
  - name: netflow
    var.input.udp.port: 2056
    var.elasticsearch.hosts: http://127.0.0.1:9200
    var.elasticsearch.ssl.enabled: false
    var.kibana.host: 127.0.0.1:5601
    var.kibana.scheme: http
    var.kibana.ssl.enabled: false
    var.kibana.ssl.verification_mode: disable

(...)
```

Try running as root again:
```bash
root@ELK:/usr/local/logstash # /usr/local/logstash/bin/logstash --modules netflow --setup
Sending Logstash logs to /usr/local/logstash/logs which is now configured via log4j2.properties
[2020-01-01T16:18:58,468][INFO ][logstash.config.source.modules] Both command-line and logstash.yml modules configurations detected. Using command-line module configuration to override logstash.yml module configuration.
[2020-01-01T16:18:58,486][WARN ][logstash.config.source.multilocal] Ignoring the 'pipelines.yml' file because modules or command line options are specified
[2020-01-01T16:18:58,497][INFO ][logstash.runner          ] Starting Logstash {"logstash.version"=>"6.8.5"}
[2020-01-01T16:18:59,419][INFO ][logstash.config.source.modules] Both command-line and logstash.yml modules configurations detected. Using command-line module configuration to override logstash.yml module configuration.
[2020-01-01T16:18:59,515][INFO ][logstash.config.modulescommon] Setting up the netflow module
[2020-01-01T16:18:59,894][WARN ][logstash.modules.kibanaclient] SSL explicitly disabled; other SSL settings will be ignored
[2020-01-01T16:19:21,538][INFO ][logstash.pipeline        ] Starting pipeline {:pipeline_id=>"module-netflow", "pipeline.workers"=>24, "pipeline.batch.size"=>125, "pipeline.batch.delay"=>50}
[2020-01-01T16:19:21,824][INFO ][logstash.outputs.elasticsearch] Elasticsearch pool URLs updated {:changes=>{:removed=>[], :added=>[http://127.0.0.1:9200/]}}
[2020-01-01T16:19:21,934][WARN ][logstash.outputs.elasticsearch] Restored connection to ES instance {:url=>"http://127.0.0.1:9200/"}
[2020-01-01T16:19:21,956][INFO ][logstash.outputs.elasticsearch] ES Output version determined {:es_version=>6}
[2020-01-01T16:19:21,960][WARN ][logstash.outputs.elasticsearch] Detected a 6.x and above cluster: the `type` event field won't be used to determine the document _type {:es_version=>6}
[2020-01-01T16:19:21,992][INFO ][logstash.outputs.elasticsearch] New Elasticsearch output {:class=>"LogStash::Outputs::ElasticSearch", :hosts=>["http://127.0.0.1:9200"]}
[2020-01-01T16:19:22,891][INFO ][logstash.filters.geoip   ] Using geoip database {:path=>"/usr/local/logstash/vendor/bundle/jruby/2.5.0/gems/logstash-filter-geoip-5.0.3-java/vendor/GeoLite2-City.mmdb"}
[2020-01-01T16:19:22,913][INFO ][logstash.filters.geoip   ] Using geoip database {:path=>"/usr/local/logstash/vendor/bundle/jruby/2.5.0/gems/logstash-filter-geoip-5.0.3-java/vendor/GeoLite2-ASN.mmdb"}
[2020-01-01T16:19:22,915][INFO ][logstash.filters.geoip   ] Using geoip database {:path=>"/usr/local/logstash/vendor/bundle/jruby/2.5.0/gems/logstash-filter-geoip-5.0.3-java/vendor/GeoLite2-City.mmdb"}
[2020-01-01T16:19:22,916][INFO ][logstash.filters.geoip   ] Using geoip database {:path=>"/usr/local/logstash/vendor/bundle/jruby/2.5.0/gems/logstash-filter-geoip-5.0.3-java/vendor/GeoLite2-ASN.mmdb"}
[2020-01-01T16:19:22,917][INFO ][logstash.filters.geoip   ] Using geoip database {:path=>"/usr/local/logstash/vendor/bundle/jruby/2.5.0/gems/logstash-filter-geoip-5.0.3-java/vendor/GeoLite2-City.mmdb"}
[2020-01-01T16:19:22,918][INFO ][logstash.filters.geoip   ] Using geoip database {:path=>"/usr/local/logstash/vendor/bundle/jruby/2.5.0/gems/logstash-filter-geoip-5.0.3-java/vendor/GeoLite2-ASN.mmdb"}
[2020-01-01T16:19:22,918][INFO ][logstash.filters.geoip   ] Using geoip database {:path=>"/usr/local/logstash/vendor/bundle/jruby/2.5.0/gems/logstash-filter-geoip-5.0.3-java/vendor/GeoLite2-City.mmdb"}
[2020-01-01T16:19:22,919][INFO ][logstash.filters.geoip   ] Using geoip database {:path=>"/usr/local/logstash/vendor/bundle/jruby/2.5.0/gems/logstash-filter-geoip-5.0.3-java/vendor/GeoLite2-ASN.mmdb"}
[2020-01-01T16:19:23,017][INFO ][logstash.pipeline        ] Pipeline started successfully {:pipeline_id=>"module-netflow", :thread=>"#<Thread:0x496d4961 run>"}
[2020-01-01T16:19:23,058][INFO ][logstash.inputs.udp      ] Starting UDP listener {:address=>"0.0.0.0:2056"}
[2020-01-01T16:19:23,096][INFO ][logstash.agent           ] Pipelines running {:count=>1, :running_pipelines=>[:"module-netflow"], :non_running_pipelines=>[]}
[2020-01-01T16:19:23,140][INFO ][logstash.inputs.udp      ] UDP listener started {:address=>"0.0.0.0:2056", :receive_buffer_bytes=>"212992", :queue_size=>"2000"}
[2020-01-01T16:19:23,435][INFO ][logstash.agent           ] Successfully started Logstash API endpoint {:port=>9600}
```

Ok, good. Wait a minute and check if the different netflow dashboards have been generated by `--setup` in Kibana from your webbrowser. Now quit `logstash` by pressing `Ctrl + c`.

PS: The `--setup` parameter will add searches, dashboards and vizualisations in Kibana for netflow module. If you run logstash with parameter `--setup` one more time, it will override your (eventual) dashboard edits concerning Netflow. 

What I discovered when specifying netflow as a module in `logstash.yml`, was that logstash is ignoring the ‘pipelines.yml’ file because modules or command line options are specified. So the pfSense syslogs broke (it was not picked up at all).

So what we want to do, is to edit the `/usr/local/logstash/modules/netflow/configuration/logstash/netflow.conf.erb` file, which has been created from the `--setup`parameter to a `*.conf file`, like the ones you now have in `/etc/logstash/conf.d/`.

```bash
root@ELK:/usr/local/logstash # cp /usr/local/logstash/modules/netflow/configuration/logstash/netflow.conf.erb /usr/local/etc/logstash/conf.d/netflow.conf
```
`netflow.conf.erb` is an ERB template so there are sections that are overwritten, in between `<%=, %>`, with real values - replace these and you will have a configuration file for netflow.

You can do that by yourself:
```bash
root@ELK:/usr/local/logstash # nano /usr/local/etc/logstash/conf.d/netflow.conf
```

or just wget this `netflow.conf`
```bash
root@ELK:/usr/local/etc/logstash/conf.d # wget https://raw.githubusercontent.com/psychogun/ELK-Stack-on-Ubuntu-for-pfSense/master/etc/logstash/conf.d/netflow.conf
```

PS: Edit this `netflow.conf`, as it is made for the Linux installation i mentioned earlier on top, so some paths are wrong.
For example, change `dictionary_path => "/usr/share/logstash/modules/netflow/configuration/logstash/dictionaries/iana_service_names_dccp.yml"` to `dictionary_path => "/usr/local/logstash/modules/netflow/configuration/logstash/dictionaries/iana_service_names_dccp.yml"`.


Remember, we do not use `pipelines.yml` (yet (??)); so let us go ahead and `cat` a new `logstash.conf` file that includes our newly created `netflow.conf` config:
```bash
root@ELK:/usr/local/etc/logstash/conf.d # cat 01-inputs.conf 10-syslog.conf 11-pfsene.conf netflow.conf 12-pfsense 30-outputs.conf > /usr/local/etc/logstash/logstash.conf
```

Edit `logstash.yml`:
```bash
root@ELK:/usr/local/etc/logstash # nano logstash.yml

(...)
path.config: /usr/local/etc/logstash/logstash.conf
(...)

(...)
#modules:
#  - name: netflow
#    var.input.udp.port: 2056
#    var.elasticsearch.hosts: http://127.0.0.1:9200
#    var.elasticsearch.ssl.enabled: false
#    var.kibana.host: 127.0.0.1:5601
#    var.kibana.scheme: http
#    var.kibana.ssl.enabled: false
#    var.kibana.ssl.verification_mode: disable
(...)
```
Try and start `logstash` again:
```bash
root@ELK:/usr/local/etc/logstash/conf.d # service logstash start
```
Was everythings OK?

```bash
root@ELK:~ # tail -f /var/log/logstash/logstash-plain.log 
[2020-01-01T16:38:24,064][INFO ][logstash.filters.geoip   ] Using geoip database {:path=>"/usr/local/logstash/vendor/bundle/jruby/2.5.0/gems/logstash-filter-geoip-5.0.3-java/vendor/GeoLite2-ASN.mmdb"}
[2020-01-01T16:38:24,064][INFO ][logstash.filters.geoip   ] Using geoip database {:path=>"/usr/local/logstash/vendor/bundle/jruby/2.5.0/gems/logstash-filter-geoip-5.0.3-java/vendor/GeoLite2-City.mmdb"}
[2020-01-01T16:38:24,065][INFO ][logstash.filters.geoip   ] Using geoip database {:path=>"/usr/local/logstash/vendor/bundle/jruby/2.5.0/gems/logstash-filter-geoip-5.0.3-java/vendor/GeoLite2-ASN.mmdb"}
[2020-01-01T16:38:24,186][INFO ][logstash.pipeline        ] Pipeline started successfully {:pipeline_id=>"main", :thread=>"#<Thread:0x45fadcd2 run>"}
[2020-01-01T16:38:24,248][INFO ][logstash.inputs.udp      ] Starting UDP listener {:address=>"0.0.0.0:2055"}
[2020-01-01T16:38:24,253][INFO ][logstash.inputs.udp      ] Starting UDP listener {:address=>"0.0.0.0:5140"}
[2020-01-01T16:38:24,310][INFO ][logstash.agent           ] Pipelines running {:count=>1, :running_pipelines=>[:main], :non_running_pipelines=>[]}
[2020-01-01T16:38:24,330][INFO ][logstash.inputs.udp      ] UDP listener started {:address=>"0.0.0.0:5140", :receive_buffer_bytes=>"42080", :queue_size=>"2000"}
[2020-01-01T16:38:24,330][INFO ][logstash.inputs.udp      ] UDP listener started {:address=>"0.0.0.0:2055", :receive_buffer_bytes=>"212992", :queue_size=>"2000"}
[2020-01-01T16:38:24,887][INFO ][logstash.agent           ] Successfully started Logstash API endpoint {:port=>9600}
```

## Managing the index lifecycle
For me, my `netflow` elasticsearch indices generates at a size of approx 500mb a day.
The same as my `logstash` indice.  If you have an unlimited storage space, this might not be a problem for you - but for me, to keep things in the clear- I would only want to keep my indices for 30 days. That way, the storage space needed would never go above 30 x (500mb + 500mb) = 30GB.


## Fault finding

Try first to see if you can upgrade anything:
```bash
root@ELK:~ # pkg update
Updating FreeBSD repository catalogue...
[ELK] Fetching meta.txz: 100%    944 B   0.9kB/s    00:01    
[ELK] Fetching packagesite.txz: 100%    6 MiB   1.3MB/s    00:05    
Processing entries: 100%
FreeBSD repository update completed. 31707 packages processed.
All repositories are up to date.
root@ELK:~ # pkg upgrade
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
Checking for upgrades (6 candidates): 100%
Processing candidates (6 candidates): 100%
The following 5 package(s) will be affected (of 0 checked):

Installed packages to be UPGRADED:
	logstash6: 6.8.5 -> 6.8.6
	libuv: 1.34.0 -> 1.34.1
	kibana6: 6.8.5 -> 6.8.6
	elasticsearch6: 6.8.5 -> 6.8.6
	ca_root_nss: 3.49 -> 3.49.1

Number of packages to be upgraded: 5

The process will require 2 MiB more space.
395 MiB to be downloaded.

Proceed with this action? [y/N]: y
```

### Elasticsearch
Is `elasticsearch` running?
```bash
root@ELK:~ # pkg install curl
root@ELK:~ # curl localhost:9200
 {
  "name" : "zG8etaX",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "KpTDT1JTYDWXYzXgQFJ9Sv",
  "version" : {
    "number" : "6.8.5",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "78990e9",
    "build_date" : "2019-11-13T20:04:24.100411Z",
    "build_snapshot" : false,
    "lucene_version" : "7.7.2",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
root@ELK:~ #  
```
#### Java heap space
```bash
root@ELK:~ # curl localhost:9200
curl: (7) Failed to connect to localhost port 9200: Connection refused
root@ELK:~ # service elasticsearch status
elasticsearch is not running.
root@ELK:~ # 
```
What does the log file say?

```bash
root@ELK:~ # tail -f /var/log/elasticsearch/elasticsearch.log 
	at org.elasticsearch.threadpool.Scheduler$ReschedulingRunnable.doRun(Scheduler.java:247) [elasticsearch-6.8.6.jar:6.8.6]
	at org.elasticsearch.common.util.concurrent.AbstractRunnable.run(AbstractRunnable.java:37) [elasticsearch-6.8.6.jar:6.8.6]
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511) [?:1.8.0_232]
	at java.util.concurrent.FutureTask.run(FutureTask.java:266) [?:1.8.0_232]
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$201(ScheduledThreadPoolExecutor.java:180) [?:1.8.0_232]
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:293) [?:1.8.0_232]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) [?:1.8.0_232]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) [?:1.8.0_232]
	at java.lang.Thread.run(Thread.java:748) [?:1.8.0_232]
Caused by: java.lang.OutOfMemoryError: Java heap space
```
Googled, and I found this [https://discuss.elastic.co/t/java-heap-space-once-in-a-while/2551/3](https://discuss.elastic.co/t/java-heap-space-once-in-a-while/2551/3) which got me to [https://www.elastic.co/guide/en/elasticsearch/guide/current/heap-sizing.html](https://www.elastic.co/guide/en/elasticsearch/guide/current/heap-sizing.html), so I'll try that.

```bash
root@ELK:/usr/local/etc/elasticsearch # nano jvm.options
# Xms represents the initial size of total heap space
# Xmx represents the maximum size of total heap space

-Xms6g
-Xmx6g
```
Then we'll restart `elasticsearch` and we'll see how it goes with 6GiB heap size.
```bash
root@ELK:/usr/local/etc/elasticsearch # service elasticsearch start
Starting elasticsearch.
root@ELK:/usr/local/etc/elasticsearch # 
```
Got:
Cannot connect to the Elasticsearch cluster
See the Kibana logs for details and try reloading the page.

I'll issue a STOP/START from FreeNAS gui.
### Logstash

```bash
root@ELK:~ # tail -f /var/log/logstash/logstash-plain.log 
[2020-01-18T13:28:29,808][WARN ][logstash.outputs.elasticsearch] Restored connection to ES instance {:url=>"http://localhost:9200/"}
[2020-01-18T13:45:08,190][WARN ][logstash.outputs.elasticsearch] Marking url as dead. Last error: [LogStash::Outputs::ElasticSearch::HttpClient::Pool::HostUnreachableError] Elasticsearch Unreachable: [http://localhost:9200/][Manticore::SocketTimeout] Read timed out {:url=>http://localhost:9200/, :error_message=>"Elasticsearch Unreachable: [http://localhost:9200/][Manticore::SocketTimeout] Read timed out", :error_class=>"LogStash::Outputs::ElasticSearch::HttpClient::Pool::HostUnreachableError"}
[2020-01-18T13:45:08,191][ERROR][logstash.outputs.elasticsearch] Attempted to send a bulk request to elasticsearch' but Elasticsearch appears to be unreachable or down! {:error_message=>"Elasticsearch Unreachable: [http://localhost:9200/][Manticore::SocketTimeout] Read timed out", :class=>"LogStash::Outputs::ElasticSearch::HttpClient::Pool::HostUnreachableError", :will_retry_in_seconds=>2}
[2020-01-18T13:45:09,593][WARN ][logstash.outputs.elasticsearch] Restored connection to ES instance {:url=>"http://localhost:9200/"}
[2020-01-18T13:45:10,196][ERROR][logstash.outputs.elasticsearch] Attempted to send a bulk request to elasticsearch, but no there are no living connections in the connection pool. Perhaps Elasticsearch is unreachable or down? {:error_message=>"No Available connections", :class=>"LogStash::Outputs::ElasticSearch::HttpClient::Pool::NoConnectionAvailableError", :will_retry_in_seconds=>4}
[2020-01-18T13:58:08,766][WARN ][logstash.outputs.elasticsearch] Marking url as dead. Last error: [LogStash::Outputs::ElasticSearch::HttpClient::Pool::HostUnreachableError] Elasticsearch Unreachable: [http://localhost:9200/][Manticore::SocketTimeout] Read timed out {:url=>http://localhost:9200/, :error_message=>"Elasticsearch Unreachable: [http://localhost:9200/][Manticore::SocketTimeout] Read timed out", :error_class=>"LogStash::Outputs::ElasticSearch::HttpClient::Pool::HostUnreachableError"}
[2020-01-18T13:58:08,767][ERROR][logstash.outputs.elasticsearch] Attempted to send a bulk request to elasticsearch' but Elasticsearch appears to be unreachable or down! {:error_message=>"Elasticsearch Unreachable: [http://localhost:9200/][Manticore::SocketTimeout] Read timed out", :class=>"LogStash::Outputs::ElasticSearch::HttpClient::Pool::HostUnreachableError", :will_retry_in_seconds=>2}
[2020-01-18T13:58:10,776][ERROR][logstash.outputs.elasticsearch] Attempted to send a bulk request to elasticsearch, but no there are no living connections in the connection pool. Perhaps Elasticsearch is unreachable or down? {:error_message=>"No Available connections", :class=>"LogStash::Outputs::ElasticSearch::HttpClient::Pool::NoConnectionAvailableError", :will_retry_in_seconds=>4}
[2020-01-18T13:58:14,794][ERROR][logstash.outputs.elasticsearch] Attempted to send a bulk request to elasticsearch, but no there are no living connections in the connection pool. Perhaps Elasticsearch is unreachable or down? {:error_message=>"No Available connections", :class=>"LogStash::Outputs::ElasticSearch::HttpClient::Pool::NoConnectionAvailableError", :will_retry_in_seconds=>8}
[2020-01-18T13:58:15,068][WARN ][logstash.outputs.elasticsearch] Restored connection to ES instance {:url=>"http://localhost:9200/"}
```
So something seems to trip our `elasticsearch`. QuePasa?


### Kibana
```bash
root@ELK:~ # service kibana status
kibana is not running.
root@ELK:~ # service kibana start
Starting kibana.
root@ELK:~ # service kibana status
kibana is running as pid 81243.
root@ELK:~ # 
```

## Authors
Mr. Johnson


## Acknowledgments
* [https://github.com/robcowart/synesis_lite_suricata](https://github.com/robcowart/synesis_lite_suricata)
* [https://github.com/evaluationcopy/pfsense-suricata-elk-docker](https://github.com/evaluationcopy/pfsense-suricata-elk-docker)
* [http://blog.dushin.net/2019/08/installing-elk-on-freenas-jail/](http://blog.dushin.net/2019/08/installing-elk-on-freenas-jail/)
* [https://unix.stackexchange.com/questions/483990/change-between-quarterly-and-latest-package-set-used-by-pkg-tool-in-freebs](https://unix.stackexchange.com/questions/483990/change-between-quarterly-and-latest-package-set-used-by-pkg-tool-in-freebs)
* [https://gradle.org](https://gradle.org)
* [https://github.com/elastic/elasticsearch/issues/44651](https://github.com/elastic/elasticsearch/issues/44651)
* [https://forums.freebsd.org/threads/howto-permanently-set-switch-shell-path-to-java-vm.71419/](https://forums.freebsd.org/threads/howto-permanently-set-switch-shell-path-to-java-vm.71419)
* [https://www.linuxsecrets.com/2923-install-java-on-freebsd](https://www.linuxsecrets.com/2923-install-java-on-freebsd)
* [https://discuss.elastic.co/t/netflow-module-launch-errors-some-ssl-related/149596/7](https://discuss.elastic.co/t/netflow-module-launch-errors-some-ssl-related/149596/7)
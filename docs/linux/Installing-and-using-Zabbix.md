---
layout: default
title: Installing and using Zabbix 
parent: Linux
nav_order: 9
---
# Installing and using Zabbix 
{: .no_toc }
This is how I installed a new instance of Zabbix on a Proxmox guest, moving from an ESXi host and along transfer the new database to the newly created guest.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}
---
## Getting started


## Prerequisites
* Ubuntu 18.04
* MySQL
* NGINX

## Install qemu-guest-agent
The qemu-guest-agent is a helper daemon, which is installed in the guest. It is used to exchange information between the host and guest, and to execute command in the guest.

In Proxmox VE, the qemu-guest-agent is used for mainly two things:

To properly shutdown the guest, instead of relying on ACPI commands or windows policies
To freeze the guest file system when making a backup (on windows, use the volume shadow copy service VSS).

```bash
administrator@zabbix:~$ sudo apt-get install qemu-guest-agent
administrator@zabbix:~$ sudo shutdown now
```
In Proxmox, go to Options and Enable by selecting `Use QEMU Guest Agent`. Start your VM again. 

## tzdata
Ensure that your timezone is correct.
```bash
merovingian@zabbix:~$ sudo dpkg-reconfigure tzdata
```


## Perform backup
Outline;
Server upgrade process

1 Stop Zabbix server

Stop Zabbix server to make sure that no new data is inserted into database.

2 Back up the existing Zabbix database

This is a very important step. Make sure that you have a backup of your database. It will help if the upgrade procedure fails (lack of disk space, power off, any unexpected problem).

3 Back up configuration files, PHP files and Zabbix binaries

Make a backup copy of Zabbix binaries, configuration files and the PHP file directory.

### Stop Zabbix server
Stop Zabbix server to make sure that no new data is inserted into database.
```bash
zabbix@linuxdog:~$ service zabbix-server stop
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-units ===
Authentication is required to stop 'zabbix-server.service'.
Authenticating as: Monitoring Tool (zabbix)
Password: 
==== AUTHENTICATION COMPLETE ===
zabbix@linuxdog:~$ 
```
```
zabbix@linuxdog:~$ service zabbix-agent stop
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-units ===
Authentication is required to stop 'zabbix-agent.service'.
Authenticating as: Monitoring Tool (zabbix)
Password: 
==== AUTHENTICATION COMPLETE ===
zabbix@linuxdog:~$ 
```

### Back up the existing Zabbix database
This is a very important step. Make sure that you have a backup of your database. It will help if the upgrade procedure fails (lack of disk space, power off, any unexpected problem).

List available databases:
```bash
zabbix@linuxdog:~$ sudo su
[sudo] password for zabbix: 
root@linuxdog:/home/zabbix# mysql
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 31209
Server version: 10.1.43-MariaDB-0ubuntu0.18.04.1 Ubuntu 18.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> SELECT table_schema "databases name", sum(data_length + index_length)/1024/1024  "DВ size in MB" FROM information_schema.TABLES GROUP BY table_schema;
+--------------------+----------------+
| databases name     | DВ size in MB  |
+--------------------+----------------+
| information_schema |     0.17187500 |
| mysql              |     0.87331867 |
| performance_schema |     0.00000000 |
| zabbix             |  1187.75000000 |
+--------------------+----------------+
4 rows in set (0.09 sec)

MariaDB [(none)]> 
```

### Back up configuration files, PHP files and Zabbix binaries
Make a backup copy of Zabbix binaries, configuration files and the PHP file directory.

Configuration files:
```bash
zabbix@linuxdog:~$ sudo mkdir /opt/zabbix-backup/
zabbix@linuxdog:~$ cp /etc/zabbix/zabbix_server.conf /opt/zabbix-backup/
zabbix@linuxdog:~$ sudo cp /etc/zabbix/zabbix_server.conf /opt/zabbix-backup/
zabbix@linuxdog:~$ sudo cp /etc/apache2/conf-enabled/zabbix.conf /opt/zabbix-backup/
zabbix@linuxdog:~$ 
```
PHP files and Zabbix binaries:
```bash
zabbix@linuxdog:~$ sudo cp -R /usr/share/zabbix/ /opt/zabbix-backup/
zabbix@linuxdog:~$ sudo cp -R /usr/share/doc/zabbix-* /opt/zabbix-backup/
```

## Installation
[https://www.zabbix.com/download?zabbix=4.4&os_distribution=ubuntu&os_version=18.04_bionic&db=mysql&ws=apache](https://www.zabbix.com/download?zabbix=4.4&os_distribution=ubuntu&os_version=18.04_bionic&db=mysql&ws=apache)
### Install Zabbix repository
```bash
# wget https://repo.zabbix.com/zabbix/4.4/ubuntu/pool/main/z/zabbix-release/zabbix-release_4.4-1+bionic_all.deb
# dpkg -i zabbix-release_4.4-1+bionic_all.deb
# apt update
```
### Install Zabbix server, frontend, agent
```bash
# apt -y install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-agent
```
### Import old database
Then we are going to import the old database in to the new ZABBIX instance. Before we do that, stop all `zabbix` processes;
```bash
root@zabbix:/home/merovingian# systemctl stop zabbix-server zabbix-agent apache2
```
Then we will create a new database, `zabbix44`, and grant all privileges to which a user named `zabbix` can connect to from `localhost`. Then we will import the old database dump to this newly created `zabbix44` database:

```bash
merovingian@zabbix:~ sudo su
[sudo] password for merovingian: 
root@zabbix:/home/merovingian# 
root@zabbix:/home/merovingian# mysql
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 112
Server version: 10.1.44-MariaDB-0ubuntu0.18.04.1 Ubuntu 18.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> create database zabbix44 character set utf8 collate utf8_bin;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> grant all privileges on zabbix44.* to zabbix@localhost identified by 'zabbix-user-password;
Query OK, 0 rows affected (0.00 sec)
MariaDB [(none)]> exit;
root@zabbix:/home/merovingian# mysql -uzabbix -p zabbix44 < /home/merovingian/oldzabbix/2020-01-23\ ZABBIX\ backup/dbdump 
Enter password: zabbix-user-password
```
Importing the old database into `zabbix44` might take some time, depending on how large the old database is.

### Configure the database for Zabbix server
Edit file `/etc/zabbix/zabbix_server.conf`:
```bash
merovingian@zabbix:~$ sudo nano /etc/zabbix/zabbix_server.conf 
[sudo] password for merovingian: 
(...)
DBName=zabbix44
(...)
DBPassword=zabbix-user-password
```

### Configure PHP for Zabbix frontend
Edit file `/etc/zabbix/apache.conf`, uncomment and set the right timezone for you.
```bash
merovingian@zabbix:~$ sudo nano /etc/zabbix/apache.conf 
 # php_value date.timezone Europe/Riga
    </IfModule>
    <IfModule mod_php7.c>
(...)
 # php_value date.timezone Europe/Riga
```

### Start Zabbix server and agent processes
Now we are goingn to start Zabbix. Before we do that, open a new terminal and let us `tail` the zabbix server log file to see what is going on:
```bash
merovingian@zabbix:~$ tail -f /var/log/zabbix/zabbix_server.log 
```
Let us start everything:
```bash
merovingian@zabbix:~$ sudo systemctl restart zabbix-server zabbix-agent apache2
```
Our `tail -f` window should now display that zabbix is performing a database upgrade. 
```bash
(...)
  3288:20200211:125834.910 completed 93% of event name update
  3288:20200211:125834.910 completed 94% of event name update
  3288:20200211:125835.442 completed 95% of event name update
  3288:20200211:125835.442 completed 96% of event name update
  3288:20200211:125835.442 completed 97% of event name update
  3288:20200211:125835.443 completed 98% of event name update
  3288:20200211:125835.443 completed 99% of event name update
  3288:20200211:125835.443 completed 100% of event name update
  3288:20200211:125835.456 event name update completed
(...)
```

Make the Zabbix server and agent processes start at system boot.

```bash
# systemctl enable zabbix-server zabbix-agent apache2
```

### Configure Zabbix frontend
Connect to your newly installed Zabbix frontend: http://server_ip_or_name/zabbix.

Remember, if you are using an old database, the GUI usernames and passwords are the same as it was.












```bash
merovingian@zabbix:~/oldzabbix/2020-01-23 ZABBIX backup$ sudo su
[sudo] password for merovingian: 
root@zabbix:/home/merovingian/oldzabbix/2020-01-23 ZABBIX backup# bzip2 -d dbdump.bz2 
root@zabbix:/home/merovingian/oldzabbix/2020-01-23 ZABBIX backup# mysql
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 53
Server version: 10.1.44-MariaDB-0ubuntu0.18.04.1 Ubuntu 18.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> SELECT table_schema "databases name", sum(data_length + index_length)/1024/1024  "DВ size in MB" FROM information_schema.TABLES GROUP BY table_schema;
+--------------------+----------------+
| databases name     | DВ size in MB  |
+--------------------+----------------+
| information_schema |     0.17187500 |
| mysql              |     0.65456867 |
| performance_schema |     0.00000000 |
| zabbix             |    14.39062500 |
+--------------------+----------------+
4 rows in set (0.02 sec)

MariaDB [(none)]> exit
root@zabbix:/home/merovingian/oldzabbix/2020-01-23 ZABBIX backup# mysql -u zabbix -p zabbix < /home/merovingian/oldzabbix/2020-01-23\ ZABBIX\ backup/dbdump 
Enter password: 
root@zabbix:/home/merovingian/oldzabbix/2020-01-23 ZABBIX backup# mysql
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 55
Server version: 10.1.44-MariaDB-0ubuntu0.18.04.1 Ubuntu 18.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> SELECT table_schema "databases name", sum(data_length + index_length)/1024/1024  "DВ size in MB" FROM information_schema.TABLES GROUP BY table_schema;
+--------------------+----------------+
| databases name     | DВ size in MB  |
+--------------------+----------------+
| information_schema |     0.17187500 |
| mysql              |     0.65456867 |
| performance_schema |     0.00000000 |
| zabbix             |   971.48437500 |
+--------------------+----------------+
4 rows in set (0.07 sec)

MariaDB [(none)]> 
```

## FreeNAS 
### Dependencies
merovingian@zabbix:/usr/bin$ sudo apt-get install snmp snmp-mibs-downloader

### Coniguration of FreeNAS

Go to FreeNAS GUI and select Services. Edit SNMP as follow:

* Location: FreeNAS
* Contact: bender@ilovebender.com
* Community: BenderNAS
* [v] SNMP v3 Support
* Username: usernameX
* Authentication Type: MD5
* Password: password1
* Privacy Protocol: DES
* Privacy Passphrase: passphrase2
* Log Level: Error

Hit SAVE. 

Copy all the mibs from `/usr/local/share/snmp/mibs/` on your FreeNAS to `/usr/share/snmp/mibs` on your Zabbix.
### Import templates
Download [https://share.zabbix.com/storage-devices/freenas/freenas-11](https://share.zabbix.com/storage-devices/freenas/freenas-11). 

Go to Configuration > Hosts and select your FreeNAS server. Go to Macros and add
* `{$SNMP_COMMUNITY} is the Community (BenderNAS)
* `{$SNMP_SECNAME}` is the Username (usernameX)
* `{$SNMP_AUTH}` is the Authentication password (password1)
* `{$SNMP_PRIV}` is the Pricacy Passphrase (passphrase2)

{$SNMP_USER} – User name used for authentication in SNMPv3 (by default “racom”)

Restart the Zabbix server.

snmpwalk -v3 -l authPriv -u rtyuih675 -a SHA -A "askdjhfz67z6asd7fgoya" -x DES -X "oisduhx7986x8ygai" 192.168.5.13:161



## Fault finding

### Stop all Zabbix processes
```bash
merovingian@zabbix:~$ sudo su
root@zabbix:/home/merovingian# mysql
root@zabbix:/home/merovingian# systemctl stop zabbix-server zabbix-agent apache2
```

### Delete a database
```bash
merovingian@zabbix:~$ sudo su
root@zabbix:/home/merovingian# mysql
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 66
Server version: 10.1.44-MariaDB-0ubuntu0.18.04.1 Ubuntu 18.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases
    -> ;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| zabbix             |
+--------------------+
4 rows in set (0.01 sec)

MariaDB [(none)]> DROP DATABASE zabbix;
Query OK, 150 rows affected (5.05 sec)

MariaDB [(none)]> show databases
    -> ;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
3 rows in set (0.00 sec)

MariaDB [(none)]> 

```

### Show version
```bash
root@zabbix:/home/merovingian# zabbix_server --version
zabbix_server (Zabbix) 4.4.5
Revision b93f5c4fc0 28 January 2020, compilation time: Jan 30 2020 11:27:03

Copyright (C) 2020 Zabbix SIA
License GPLv2+: GNU GPL version 2 or later <http://gnu.org/licenses/gpl.html>.
This is free software: you are free to change and redistribute it according to
the license. There is NO WARRANTY, to the extent permitted by law.

This product includes software developed by the OpenSSL Project
for use in the OpenSSL Toolkit (http://www.openssl.org/).

Compiled with OpenSSL 1.1.0g  2 Nov 2017
Running with OpenSSL 1.1.1  11 Sep 2018
root@zabbix:/home/merovingian# 
```

### Unable to connect  
Is `zabbix.conf.php` pointing to the correct database?

```bash
merovingian@zabbix:/etc/zabbix/web$ ls -alh
total 12K
drwxr-xr-x 2 www-data root     4.0K Feb 11 14:41 .
drwxr-xr-x 4 root     root     4.0K Feb 11 14:35 ..
-rw-r--r-- 1 www-data www-data  431 Feb 11 14:36 zabbix.conf.php
merovingian@zabbix:/etc/zabbix/web$ sudo nano zabbix.conf.php 
```



bzip2 -d dbdump.bz2 

## Acknowledgments
* [https://www.zabbix.com/documentation/current/manual/config/items/itemupdate](https://www.zabbix.com/documentation/current/manual/config/items/itemupdate)
* [https://www.zabbix.com/forum/zabbix-troubleshooting-and-problems/27121-unable-to-get-info-from-snmp-snmp_build-error-in-log-file](https://www.zabbix.com/forum/zabbix-troubleshooting-and-problems/27121-unable-to-get-info-from-snmp-snmp_build-error-in-log-file)
* [https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/security-name-edit-snmp-target-parameters.html](https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/security-name-edit-snmp-target-parameters.html)
* [https://www.webnms.com/simulator/help/sim_network/netsim_conf_snmpv3.html](https://www.webnms.com/simulator/help/sim_network/netsim_conf_snmpv3.html)
* [https://www.racom.eu/eng/products/m/ripex/app/snmp/nms_zabbix.html](https://www.racom.eu/eng/products/m/ripex/app/snmp/nms_zabbix.html)
* [https://www.linickx.com/snmpwalk-v3-and-snmpget-v3-examples](https://www.linickx.com/snmpwalk-v3-and-snmpget-v3-examples)
* [https://www.zabbix.com/documentation/2.0/manual/config/items/itemtypes/snmp](https://www.zabbix.com/documentation/2.0/manual/config/items/itemtypes/snmp)
* [https://share.zabbix.com/zabbix-tools-and-utilities/zabbix-template-for-lld-snmpv3-cisco-network-devices-switches-and-routers-with-authentication-and-aes-des-encryption](https://share.zabbix.com/zabbix-tools-and-utilities/zabbix-template-for-lld-snmpv3-cisco-network-devices-switches-and-routers-with-authentication-and-aes-des-encryption)
* [https://www.wikihow.com/Delete-a-MySQL-Database](https://www.wikihow.com/Delete-a-MySQL-Database)
* [https://www.zabbix.com/forum/zabbix-troubleshooting-and-problems/51291-database-upgrade-failed-from-3-2-to-3-4-failed](https://www.zabbix.com/forum/zabbix-troubleshooting-and-problems/51291-database-upgrade-failed-from-3-2-to-3-4-failed)
* [https://stackoverflow.com/questions/4546778/how-can-i-import-a-database-with-mysql-from-terminal](https://stackoverflow.com/questions/4546778/how-can-i-import-a-database-with-mysql-from-terminal)
* [https://www.cyberciti.biz/faq/linuxunix-how-to-extract-and-decompress-a-bz2-tbz2-file/](https://www.cyberciti.biz/faq/linuxunix-how-to-extract-and-decompress-a-bz2-tbz2-file/)
* [https://www.zabbix.com/documentation/3.2/manual/installation/upgrade?s[]=backup&s[]=database](https://www.zabbix.com/documentation/3.2/manual/installation/upgrade?s[]=backup&s[]=database)
* [https://www.zabbix.com/forum/zabbix-troubleshooting-and-problems/383915-zabbix-s-db-mysql-export-and-import](https://www.zabbix.com/forum/zabbix-troubleshooting-and-problems/383915-zabbix-s-db-mysql-export-and-import)
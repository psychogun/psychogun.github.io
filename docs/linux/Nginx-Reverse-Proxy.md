---
layout: default
title: Nginx Reverse Proxy
parent: Linux
nav_order: 15
---
# Nginx Reverse Proxy
{: .no_toc }
This is how I used Nginx Reverse Proxy for accessing my internal services on the web. 

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}
---
## Getting started

## Prerequisites
* Proxmox 6.1
* Ubuntu 18.04
* MariaDB 10.3.23

### Set your time zone
```bash
igor@nginx:~$ date
Sat 11 Jan 21:22:53 GMT 2020
igor@nginx:~$ sudo dpkg-reconfigure tzdata

Current default time zone: 'Europe/Paris'
Local time is now:      Sat Jan 11 22:24:07 CET 2020.
Universal Time is now:  Sat Jan 11 21:24:07 UTC 2020.

igor@nginx:~$ $ date
Sat 11 Jan 22:24:16 CET 2020
```

### Disable IPv6
```bash
igor@nginx:~$ sudo nano /etc/default/grub
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
igor@nginx:~$ sudo update-grub
```

### Change NTP server
```bash
igor@nginx:~$ nano /etc/systemd/timesyncd.conf 
(...)
[Time]
#NTP=
(...)
```
Change to
```bash
(...)
[Time]
NTP=192.168.44.1 pool.ntp.org
(...)
```
Restart NTP service
```bash
igor@nginx:~$ sudo systemctl restart systemd-timesyncd
```
### qemu-guest-agent
```bash
igor@nginx:~$ sudo apt update
igor@nginx:~$ sudo apt install qemu-guest-agent
```
Issue `sudo shutdown now`. 

### Second interface Add a second interface in proxmox. VLAN tag 22. 
Enable VLAN interface 22 in pfsense, activate DHCP server.
Change / add network (T) and native in UniFi Network. 

Add extra interface on the proxmox guest host in Proxmox web ui, remember the VLAN tag. 
Start the VM. 

List interfaces by issuing `ip address`. 

To enable interface `ens19` and add DHCP on this interface, edit `00-installer-config.yaml`:
```bash
igor@nginx:~$ sudo nano /etc/netplan/00-installer-config.yaml 
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens18:
      dhcp4: true
    ens19:
      dhcp4: true
      dhcp4-overrides:
         route-metric: 800
#      routes:
#      - to: 0.0.0.0/0
#        via: 192.168.82.1
#        metric: 800
  version: 2

```
Issue `sudo reboot now`.

#### Change metric
```bash
igor@nginx:~$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.82.1    0.0.0.0         UG    100    0        0 ens19
0.0.0.0         192.168.26.1    0.0.0.0         UG    100    0        0 ens18
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.26.0    0.0.0.0         255.255.255.0   U     0      0        0 ens18
192.168.26.1    0.0.0.0         255.255.255.255 UH    100    0        0 ens18
192.168.82.0    0.0.0.0         255.255.255.248 U     0      0        0 ens19
192.168.82.1    0.0.0.0         255.255.255.255 UH    100    0        0 ens19
igor@nginx:~$ 
```



## Portainer prerequisites
We install the necessary packages to be able to install Docker:
```bash
igor@nginx:~$ sudo apt install apt-transport-https ca-certificates curl software-properties-common
```
We add the official Docker GPG key:
```bash
igor@nginx:~$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```
We activate Docker repository and update it:
```bash
igor@nginx:~$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
igor@nginx:~$ sudo apt update
```
We install the latest Docker version:
```bash
igor@nginx:~$ sudo apt install docker-ce
```

## Portainer installation
As we mentioned at the beginning of this article, installing Portainer is very simple since it works in a Docker container, for this we will execute:
```bash
igor@nginx:~$ sudo docker volume create portainer_data
igor@nginx:~$ sudo docker run -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer
```

### Admin user
Create an admin user through interface http://ip-adress:90000, after which you have opted for `Connect local`.

### Make portainer start at boot
Go to Containers, select the portainer container and under `Container details`, select `Restart policies` and use `Always` and press `Update`. 
Connect locally. 


## Nginx Proxy Manager prerequisites
### MariaDB
First add the MariaDB GPG key to your system using the following command:
```bash
igor@nginx:/usr/local/etc$ sudo apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xF1656F24C74CD1D8
```
Once the key is imported, add the MariaDB repository with:
```bash
igor@nginx:/usr/local/etc$ sudo add-apt-repository 'deb [arch=amd64,arm64,ppc64el] http://ftp.utexas.edu/mariadb/repo/10.3/ubuntu bionic main'
```
To be able to install packages from the MariaDB repository you’ll need to update the packages list:
```bash
igor@nginx:/usr/local/etc$ sudo apt update
```
Now that the repository is added install the MariaDB package with:
```bash
igor@nginx:/usr/local/etc$ sudo apt install mariadb-server
```
The MariaDB service will start automatically, to verify it type:
```bash
igor@nginx:/usr/local/etc$ sudo systemctl status mariadb
```
And print the MariaDB server version, with:
```bash
igor@nginx:/usr/local/etc$ mysql -V
mysql  Ver 15.1 Distrib 10.3.23-MariaDB, for debian-linux-gnu (x86_64) using readline 5.2
```
#### Securing MariaDB
Run the mysql_secure_installation command to improve the security of the MariaDB installation:
```bash
sudo mysql_secure_installation
```
Allow connections from remote host:
```bash
igor@nginx:/etc/mysql/mariadb.conf.d$ sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf 
bind-address = 172.17.0.1
```
The `bind-address` above is the `docker0` interface (`ifconfig`).

### Create a DB for NPM
```bash
igor@nginx:~$ sudo mysql -u root
MariaDB [(none)]> CREATE DATABASE npm;
Query OK, 1 row affected (0.000 sec)

MariaDB [(none)]> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| npm                |
| performance_schema |
+--------------------+
4 rows in set (0.000 sec)

MariaDB [(none)]> 
```
### Create a new MariaDB user;
```bash
MariaDB [(none)]> CREATE USER 'npm'@localhost IDENTIFIED BY 'password1';
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]> SELECT User FROM mysql.user;
+------------------+
| User             |
+------------------+
| debian-sys-maint |
| npm              |
| root             |
+------------------+
3 rows in set (0.000 sec)

MariaDB [(none)]> 
```
### Grant privileges to MariaDB user for db 'npm'
Let us make sure our user can connect from the NGINX docker from ip-address `172.17.0.3`:
```bash
MariaDB [(none)]> GRANT ALL PRIVILEGES ON npm.* TO 'npm'@'172.17.0.3' IDENTIFIED BY 'password';
Query OK, 0 rows affected (0.000 sec)
```
It’s crucial to refresh the privileges once new ones have been awarded with the command:
```bash
MariaDB [(none)]> FLUSH PRIVILEGES;
```
The user you have created now has full privileges and access to the specified database and tables.
Once you have completed this step, you can verify the new user1 has the right permissions by using the following statement:
```bash
MariaDB [(none)]> SHOW GRANTS FOR 'npm'@localhost;
+------------------------------------------------------------------------------------------------------------+
| Grants for npm@localhost                                                                                   |
+------------------------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO `npm`@`localhost` IDENTIFIED BY PASSWORD '*D28D32D11DCDDACF77E9456485C8BDF9482D84B2' |
| GRANT ALL PRIVILEGES ON `npm`.* TO `npm`@`localhost`                                                       |
+------------------------------------------------------------------------------------------------------------+
2 rows in set (0.000 sec)
```

### config.json
```bash
igor@nginx:~$ cd /usr/local/etc
igor@nginx:/usr/local/etc$ sudo mkdir npm
igor@nginx:/usr/local/etc$ cd npm
igor@nginx:/usr/local/etc/npm$ sudo nano config.json
{
  "database": {
    "engine": "mysql",
    "host": "ip-adress-to-mariadb-docker-interface-172.17.0.1",
    "name": "npm",
    "user": "npm",
    "password": "password1",
    "port": 3306
  }
}
```

### Deploy Nginx Proxy Manager
In Portainer, go to `Container` and select `+ Add container`. 

* Name: nginx-proxy-manager
* Image: docker.io jc21/nginx-proxy-manager
* Manual network port publishing
** host 80 > container 80
** host 443 >  container 443
** host 81 > container 81  
* Env: add environment variable: DISABLE_IPV6 true 
* Restart policy: Always 
* Volumes `map additional volumes`:
** container: /app/config/production.json
** host: /usr/local/etc/npm/config.json

Optional:
** container: /data
**  host: /usr/local/etc/npm/data

** container: letsencrypt
** host: /usr/local/etc/npm/letsencrypt

Make it writable. 


Select `Deploy the container`. 

When your docker container is running, connect to it on port 81 for the admin interface. Sometimes this can take a little bit because of the entropy of keys.

http://127.0.0.1:81

Default Admin User:

Email:    admin@example.com
Password: changeme



After a while, stop the container.


Add Let's Encrypt Certificate - use freedns network name



igor@nginx:~$ sudo ifconfig ens19 up
igor@nginx:~$ sudo dhclient ens19
cmp: EOF on /tmp/tmp.30tz3S1FTt which is empty


## Auto update container
```bash
igor@nginx:~$ sudo docker pull containrrr/watchtower
Using default tag: latest
latest: Pulling from containrrr/watchtower
3ed6cec7d4d1: Pull complete 
c0706b5a6b44: Pull complete 
6eae408ad811: Pull complete 
Digest: sha256:ff3ba4f421801c07754150aee4c03fdde26a0a9783d36021e81c7097d6842f25
Status: Downloaded newer image for containrrr/watchtower:latest
docker.io/containrrr/watchtower:latest
igor@nginx:~$ 
```

Update all containers:
```bash
igor@nginx:~$ sudo docker run -d --restart=always --name watchtower -v /var/run/docker.sock:/var/run/docker.sock containrrr/watchtower --stop-timeout 300s
74d839c3459e3b2153500b2f2dcc3084caa67106a521501c600846b422b8587f
```


## Fault finding
### Packets out of order
```bash
[8/18/2020] [9:17:51 PM] [Global   ] › ✖  error     Packets out of order. Got: 1 Expected: 0
```

Check the IP adress of the NGINX docker you have created- perhaps it has changed? GRANT ALL PRIVILEGES on the database from the new IP address:
```bash
igor@nginx:/usr/local/etc/npm$ sudo mariadb 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 529
Server version: 10.5.6-MariaDB-1:10.5.6+maria~focal mariadb.org binary distribution

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.


MariaDB [(none)]> GRANT ALL PRIVILEGES ON npm.* TO 'npm'@'172.17.0.3' IDENTIFIED BY 'password';
Query OK, 0 rows affected (0.000 sec)
```


### Set the clock?

[https://support.rackspace.com/how-to/mysql-connect-to-your-database-remotely/](https://support.rackspace.com/how-to/mysql-connect-to-your-database-remotely/)


## Authors
Mr. Johnson



## Acknowledgments
* [https://hub.docker.com/r/jc21/nginx-proxy-manager/tags](https://hub.docker.com/r/jc21/nginx-proxy-manager/tags)
* [https://clouding.io/hc/en-us/articles/360010398219-Install-Portainer-on-Ubuntu-18-04](https://clouding.io/hc/en-us/articles/360010398219-Install-Portainer-on-Ubuntu-18-04)
* [https://nginxproxymanager.com/advanced-config/](https://nginxproxymanager.com/advanced-config/)
* [https://linuxize.com/post/how-to-install-mariadb-on-ubuntu-18-04/](https://linuxize.com/post/how-to-install-mariadb-on-ubuntu-18-04/)
* [https://computingforgeeks.com/how-to-solve-error-1524-hy000-plugin-unix_socket-is-not-loaded-mysql-error-on-debian-ubuntu/](https://computingforgeeks.com/how-to-solve-error-1524-hy000-plugin-unix_socket-is-not-loaded-mysql-error-on-debian-ubuntu/)
* [https://support.plesk.com/hc/en-us/articles/360009592400--BUG-MariaDB-10-3-error-after-upgrade-Plugin-unix-socket-is-not-loaded](https://support.plesk.com/hc/en-us/articles/360009592400--BUG-MariaDB-10-3-error-after-upgrade-Plugin-unix-socket-is-not-loaded)
* [https://phoenixnap.com/kb/how-to-create-mariadb-user-grant-privileges](https://phoenixnap.com/kb/how-to-create-mariadb-user-grant-privileges)
* [https://mariadb.com/kb/en/configuring-mariadb-for-remote-client-access/](https://mariadb.com/kb/en/configuring-mariadb-for-remote-client-access/
* [https://webdock.io/en/docs/how-guides/how-enable-remote-access-your-mariadbmysql-database](https://webdock.io/en/docs/how-guides/how-enable-remote-access-your-mariadbmysql-database)
* [https://serverfault.com/questions/483339/changing-host-permissions-for-mysql-users](https://serverfault.com/questions/483339/changing-host-permissions-for-mysql-users)
* [https://bugs.launchpad.net/ubuntu/+source/nplan/+bug/1743200](https://bugs.launchpad.net/ubuntu/+source/nplan/+bug/1743200)
* [https://superuser.com/questions/331720/how-do-i-set-the-priority-of-network-connections-in-ubuntu](https://superuser.com/questions/331720/how-do-i-set-the-priority-of-network-connections-in-ubuntu)
* [https://netplan.io/examples](https://netplan.io/examples)
* [https://www.digitalocean.com/community/tutorials/how-to-install-mariadb-on-ubuntu-20-04](https://www.digitalocean.com/community/tutorials/how-to-install-mariadb-on-ubuntu-20-04)
* [https://www.cyberciti.biz/faq/how-to-delete-remove-user-account-in-mysql-mariadb/](https://www.cyberciti.biz/faq/how-to-delete-remove-user-account-in-mysql-mariadb/)
* [https://containrrr.dev/watchtower/arguments/](https://containrrr.dev/watchtower/arguments/)

---
layout: default
title: How to install OpenVPN server in a FreeNAS iocage jail
parent: FreeNAS
nav_order: 11
---

# How to install OpenVPN server in a FreeNAS iocage jail
{: .no_toc }
I wanted to access the Internet safely and securely from my smartphone or laptop when I am connected to an untrusted network such as the WiFi of a hotel or coffee shop. A Virtual Private Network (VPN) allows you to traverse untrusted networks privately and securely as if you were on a private network. The traffic emerges from the VPN server and continues its journey to the destination. 

## Table of contents
{: .no_toc .text-delta }
1. TOC
{:toc}
---

## Getting started
Requirements:

FreeNAS User with ssh access and sudo
SSH Client ( Putty for Windows and Terminal for MAC )
Admin access to the router where FreeNAS exists
Own domain or domain updated by DDNS or a static IP
Please follow this step by step tutorial before ask for help

Relevant data to use later in this tutorial ( use your own, this is just for reference )
Home Network: 192.168.1.0/24 ( LAN where is your FreeNAS )
NAT Network: 10.8.0.0/24 ( virtual LAN between VPN clients and your LAN )
Domain: nas.mydomain.com
VPN Server Port: 1194 UDP
VPN Outside Access Port: 443 UDP
Certificate Authority Password: Password1
Bibi40k Client Certificate Password: Password2

### Prerequisites
* FreeNAS 11.3-r5

## Create the jail
Remember to go in to properties of the jail in FreeNAS GUI and Edit > Custom Properties and check `v allow_tun`.
Also under Basic Properties, ensure that the jail `Autostarts`.

## Update and Upgrade
```bash
root@Mbape:~ # pkg update 
root@Mbape:~ # pkg upgrade
```

## Install OpenVPN
```bash
root@Mbape:~ # pkg install openvpn
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
The following 4 package(s) will be affected (of 0 checked):

New packages to be INSTALLED:
	openvpn: 2.4.8
	easy-rsa: 3.0.6
	lzo2: 2.10_1
	liblz4: 1.9.2,1

Number of packages to be installed: 4

The process will require 3 MiB more space.
783 KiB to be downloaded.

Proceed with this action? [y/N]: y
```

## Set up the CA directory
OpenVPN is an TLS/SSL VPN. This means that it utilizes certificates in order to encrypt traffic between the server and clients. In order to issue trusted certificates, we will need to set up our own simple certificate authority (CA).

### Create directories for OpenVPN
```bash
root@Mbape:~ # mkdir /usr/local/etc/openvpn /usr/local/etc/openvpn/keys
```

### Copy necessary files
```bash
root@Mbape:~ # cp /usr/local/share/examples/openvpn/sample-config-files/server.conf /usr/local/etc/openvpn/openvpn.conf
root@Mbape:~ # cp -r /usr/local/share/e
easy-rsa/ examples/ 
root@Mbape:~ # cp -r /usr/local/share/easy-rsa /usr/local/etc/openvpn/easy-rsa
```

### Edit /usr/local/etc/openvpn/easy-rsa/vars
```bash
root@Mbape:~ # cd /usr/local/etc/openvpn/easy-rsa
root@Mbape:/usr/local/etc/openvpn/easy-rsa # nano vars

#set_var EASYRSA_REQ_COUNTRY    "US"
#set_var EASYRSA_REQ_PROVINCE   "California"
#set_var EASYRSA_REQ_CITY       "San Francisco"
#set_var EASYRSA_REQ_ORG        "Copyleft Certificate Co"
#set_var EASYRSA_REQ_EMAIL      "me@example.net"
#set_var EASYRSA_REQ_OU         "My Organizational Unit"

#set_var EASYRSA_CA_EXPIRE 3650
#set_var EASYRSA_CERT_EXPIRE 3650
```

### Generate keys
```bash
root@Mbape:/usr/local/etc/openvpn/easy-rsa # ./easyrsa.real init-pki

Note: using Easy-RSA configuration from: ./vars

init-pki complete; you may now create a CA or requests.
Your newly created PKI dir is: /usr/local/etc/openvpn/easy-rsa/pki
```


### Build Certificate Authority 
```bash
root@Mbape:/usr/local/etc/openvpn/easy-rsa # ./easyrsa.real build-ca

Note: using Easy-RSA configuration from: ./vars

Using SSL: openssl OpenSSL 1.0.2s-freebsd  28 May 2019

Enter New CA Key Passphrase: 
Re-Enter New CA Key Passphrase: 
Generating RSA private key, 2048 bit long modulus
............+++++
..........................................................................................................+++++
e is 65537 (0x10001)
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [Easy-RSA CA]:

CA creation complete and you may now import and sign cert requests.
Your new CA certificate file for publishing is at:
/usr/local/etc/openvpn/easy-rsa/pki/ca.crt

root@Mbape:/usr/local/etc/openvpn/easy-rsa # 
```

### Build Server Certificates
```bash
root@Mbape:/usr/local/etc/openvpn/easy-rsa # ./easyrsa.real build-server-full openvpn-server nopass 

Note: using Easy-RSA configuration from: ./vars

Using SSL: openssl OpenSSL 1.0.2s-freebsd  28 May 2019
Generating a RSA private key
..+++++
............+++++
writing new private key to '/usr/local/etc/openvpn/easy-rsa/pki/private/openvpn-server.key.gbbdbQJn8T'
-----
Using configuration from /usr/local/etc/openvpn/easy-rsa/pki/safessl-easyrsa.cnf
Enter pass phrase for /usr/local/etc/openvpn/easy-rsa/pki/private/ca.key:
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'openvpn-server'
Certificate is to be certified until Dec 22 19:20:49 2020 GMT (360 days)

Write out database with 1 new entries
Data Base Updated
root@Mbape:/usr/local/etc/openvpn/easy-rsa # 
```
### Build Client Certificate 
Use unique name for each certificate, use Samba with Password2 and authorize with Password1

```bash
root@Mbape:/usr/local/etc/openvpn/easy-rsa #  ./easyrsa.real build-client-full Client

Note: using Easy-RSA configuration from: ./vars

Using SSL: openssl OpenSSL 1.0.2s-freebsd  28 May 2019
Generating a RSA private key
...........................................................................................................................................+++++
......................................+++++
writing new private key to '/usr/local/etc/openvpn/easy-rsa/pki/private/Client.key.41CnzKqXby'
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
Using configuration from /usr/local/etc/openvpn/easy-rsa/pki/safessl-easyrsa.cnf
Enter pass phrase for /usr/local/etc/openvpn/easy-rsa/pki/private/ca.key:
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'Client'
Certificate is to be certified until Dec 22 19:25:31 2020 GMT (360 days)

Write out database with 1 new entries
Data Base Updated
root@Mbape:/usr/local/etc/openvpn/easy-rsa # 
```

### Generate Diffie Hellman Parameters
( /usr/local/etc/openvpn/easy-rsa/pki/dh.pem )
```bash
root@Mbape:/usr/local/etc/openvpn/easy-rsa # ./easyrsa.real gen-dh

Note: using Easy-RSA configuration from: ./vars

Using SSL: openssl OpenSSL 1.0.2s-freebsd  28 May 2019
Generating DH parameters, 2048 bit long safe prime, generator 2
This is going to take a long time
...........................................................................................................................................+...........................................................................................................................................+............................................................................................................................................................................+............+......................................+...........+....................................+......................................+...............................................................................................+...........................................................................................................+..........................................................................................................................................................................+..................+..............................................................................+......................................................................................................+...................................................................................................................+............................................................................................................++*++*++*++*

DH parameters of size 2048 created at /usr/local/etc/openvpn/easy-rsa/pki/dh.pem

root@Mbape:/usr/local/etc/openvpn/easy-rsa # 
```


### Generate the TA key
```bash
root@Mbape:/usr/local/etc/openvpn/easy-rsa # openvpn --genkey --secret ta.key
```

### Copy Keys Together
```bash
root@Mbape:/usr/local/etc/openvpn/easy-rsa # cp pki/dh.pem pki/ca.crt pki/issued/openvpn-server.crt pki/private/openvpn-server.key /usr/local/etc/openvpn/keys/
root@Mbape:/usr/local/etc/openvpn/easy-rsa # cp ta.key /usr/local/etc/openvpn/keys/
root@Mbape:/usr/local/etc/openvpn/easy-rsa # cp pki/issued/Client.crt pki/private/Client.key /usr/local/etc/openvpn/keys/
```

### Configure OpenVPN
Edit `/usr/local/etc/openvpn/openvpn.conf`:
```bash
root@Mbape:/usr/local/etc/openvpn/easy-rsa # cd /usr/local/etc/openvpn
root@Mbape:/usr/local/etc/openvpn # nano openvpn.conf

(...)
# Which TCP/UDP port should OpenVPN listen on?
# If you want to run multiple OpenVPN instances
# on the same machine, use a different port
# number for each one.  You will need to
# open up this port on your firewall.
port 1195

# TCP or UDP server?
;proto tcp
proto udp4
(...)

(...)
# Any X509 key management system can be used.
# OpenVPN can also use a PKCS #12 formatted key file
# (see "pkcs12" directive in man page).
ca /usr/local/etc/openvpn/keys/ca.crt
cert /usr/local/etc/openvpn/keys/openvpn-server.crt
key /usr/local/etc/openvpn/keys/openvpn-server.key # This file should be kept secret

# Diffie hellman parameters.
# Generate your own with:
# openssl dhparam -out dh2048.pem 2048
dh /usr/local/etc/openvpn/keys/dh.pem
(...)

(...)
# Configure server mode and supply a VPN subnet
# for OpenVPN to draw client addresses from.
# The server will take 10.8.0.1 for itself,
# the rest will be made available to clients.
# Each client will be able to reach the server
# on 10.8.0.1. Comment this line out if you are
# ethernet bridging. See the man page for more info.
server 10.10.0.0 255.255.255.0
(...)

(...)
# Push routes to the client to allow it
# to reach other private subnets behind
# the server. Remember that these
# private subnets will also need
# to know to route the OpenVPN client
# address pool (10.8.0.0/255.255.255.0)
# back to the OpenVPN server.
;push "route 192.168.10.0 255.255.255.0"
push "route 192.168.50.0 255.255.255.0"
(...)

(...)
# The second parameter should be '0'
# on the server and '1' on the clients.
tls-auth /usr/local/etc/openvpn/keys/ta.key 0 # This file is secret
remote-cert-tls client
(...)

(...)
# Enable compression on the VPN link and push the
# option to the client (v2.4+ only, for earlier
# versions see below)
compress lz4-v2
push "compress lz4-v2"
(...)

(...)
# daemon's privileges after initialization.
# 
# You can uncomment this out on
# non-Windows systems.
user nobody
group nobody
(...)
```

## Server NAT Configuration
Create `/usr/local/etc/ipfw.rules`:
```bash
root@Mbape:/usr/local/etc/openvpn # nano /usr/local/etc/ipfw.rules
#!/bin/sh
EPAIR=$(/sbin/ifconfig -l | tr " " "\n" | /usr/bin/grep epair)
ipfw -q -f flush
ipfw -q nat 1 config if ${EPAIR}
ipfw -q add nat 1 all from 10.10.0.0/24 to any out via ${EPAIR}
ipfw -q add nat 1 all from any to any in via ${EPAIR}

TUN=$(/sbin/ifconfig -l | tr " " "\n" | /usr/bin/grep tun)
ifconfig ${TUN} name tun0
```

## Startup script
Edit `/etc/rc.conf` and add text at the end of the file:
```bash
root@Mbape:/usr/local/etc/openvpn # nano /etc/rc.conf
# OpenVPN
openvpn_enable="YES"
openvpn_if="tun"
openvpn_configfile="/usr/local/etc/openvpn/openvpn.conf"
openvpn_dir="/usr/local/etc/openvpn/"
cloned_interfaces="tun"
gateway_enable="YES"
firewall_enable="YES"
firewall_script="/usr/local/etc/ipfw.rules"
```

## Setup Logging
Edit `/etc/syslog.conf`:
```bash
root@Mbape:/usr/local/etc/openvpn # nano /etc/syslog.conf

(...)
!ppp
*.*	/var/log/ppp.log
!openvpn
*.*	/var/log/openvpn.log
!*
include                                         /etc/syslog.d
include                                         /usr/local/etc/syslog.d
```

## Setup log rotation 
Edit `/etc/newsyslog.conf`:
```bash
root@Mbape:/usr/local/etc/openvpn # nano /etc/newsyslog.conf

(...)
/var/log/xferlog                        600  7     100  *     JC

/var/log/openvpn.log                    600 30     *    @T00  ZC

<include> /etc/newsyslog.conf.d/*
<include> /usr/local/etc/newsyslog.conf.d/*
```

In FreeNAS gui, select your OpenVPN jail and press STOP. Then start it again by pressing START.

## Verify
SSH to your FreeNAS box and make some checks

### ipfw
```bash
root@Mbape:~ # ipfw list
00100 nat 1 ip from 10.6.0.0/24 to any out via epair0b
00200 nat 1 ip from any to any in via epair0b
65535 allow ip from any to any
```

### Sockstat
```bash
root@Mbape:~ # sockstat -4 -l
USER     COMMAND    PID   FD PROTO  LOCAL ADDRESS         FOREIGN ADDRESS      
root@Mbape:~ # 
```

## Run as unpriviledged user
Some misguided programs or guides suggest that this user `nobody should be used for untrusted program execution or handling untrusted data. This is bad advice. Services should have their own, dedicated, user account.

Hm.

## Client configuration
### Export files
```bash
root@Mbape:~ # cd /usr/local/etc/openvpn/keys
root@Mbape:/usr/local/etc/openvpn/keys # tar cvf Client.tar ca.crt Client.crt Client.key ta.key
a ca.crt
a Client.crt
a Client.key
a ta.key
root@Mbape:/usr/local/etc/openvpn/keys # ls -alh
root@Mbape:/usr/local/etc/openvpn/keys # ls -alh
total 75
drwxr-xr-x  2 root  wheel    10B Dec 28 23:16 .
drwxr-xr-x  4 root  wheel     7B Dec 28 22:17 ..
-rw-r--r--  1 root  wheel    12K Dec 28 23:16 Client.tar
root@Mbape:/usr/local/etc/openvpn/keys # 
```
### Encrypt the files
Install gpg:
```bash
root@Mbape:/usr/local/etc/openvpn/keys # pkg install gnupg
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
The following 21 package(s) will be affected (of 0 checked):

New packages to be INSTALLED:
	gnupg: 2.2.17_2
	pinentry: 1.1.0_5
	pinentry-tty: 1.1.0
	libgpg-error: 1.36
	libassuan: 2.5.3
	libksba: 1.3.5_1
	libgcrypt: 1.8.5
	gnutls: 3.6.10
	trousers: 0.3.14_2
	tpm-emulator: 0.7.4_2
	gmp: 6.1.2_1
	p11-kit: 0.23.17
	libtasn1: 4.14
	ca_root_nss: 3.47.1
	libffi: 3.2.1_3
	nettle: 3.5.1_1
	libidn2: 2.2.0
	libunistring: 0.9.10_1
	readline: 8.0.0
	npth: 1.6
	sqlite3: 3.29.0

Number of packages to be installed: 21

The process will require 50 MiB more space.
12 MiB to be downloaded.

Proceed with this action? [y/N]: 
```
```bash
root@Mbape:/usr/local/etc/openvpn/keys # gpg -c Client.tar
gpg: Warning: using insecure memory!
gpg: problem with the agent: End of file
gpg: error creating passphrase: Operation cancelled
gpg: symmetric encryption of 'Client.tar' failed: Operation cancelled
root@Mbape:/usr/local/etc/openvpn/keys # 
```
```bash
root@Mbape:/usr/local/etc/openvpn/keys # echo 'allow-loopback-pinentry' >> ~/.gnupg/gpg-agent.conf
root@Mbape:/usr/local/etc/openvpn/keys # gpg --pinentry-mode loopback -c Client.tar
gpg: Warning: using insecure memory!
Enter passphrase: passphrase-for-encrypted-file
```
If you have configured e-mail on your host, you can use `mpack` to send the files:
```bash
root@Mbape:/usr/local/etc/openvpn/keys # service sendmail onestart
Starting sendmail.
Starting sendmail_msp_queue.
root@Mbape:/usr/local/etc/openvpn/keys # mpack -s "Subject of the e-mail" Client.tar.gpg mbape@gmail.com
```

Click the Viscosity icon and select `Preferences`, and then `New Connection`.
### General
Connection: My Connection
Remote Server: remote-ip-address
Port: 1195

Method: Protocol: UDP
Device: tun

### Options
Type: SSL/TLS Client


### Advanced
Extra OpenVPN 
key-direction 1

```bash
root@Mbape:/usr/local/etc/openvpn # 
```

## Fault finding
Stop the service and start `openvpn` manually:

```bash
root@Mbape:~ # service openvpn stop
Stopping openvpn.
Waiting for PIDS: 12989.
root@Mbape:~ # openvpn --config /usr/local/etc/openvpn/openvpn.conf
```

Tail the log:

```bash
root@Mbape:~ # tail -f /var/log/openvpn.log
```
## Authors
Mr. Johnson

## Acknowledgments
* [https://technofaq.org/posts/2017/08/a-to-z-of-a-secure-hardened-vanilla-openvpn-setup-for-debian-stretch-gnulinux/](https://technofaq.org/posts/2017/08/a-to-z-of-a-secure-hardened-vanilla-openvpn-setup-for-debian-stretch-gnulinux/)
* [https://www.kirkg.us/posts/openvpn-server-in-a-freenas-11-jail/](https://www.kirkg.us/posts/openvpn-server-in-a-freenas-11-jail/)
* [https://www.digitalocean.com/community/tutorials/how-to-set-up-an-openvpn-server-on-ubuntu-16-04](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-openvpn-server-on-ubuntu-16-04)
* [https://www.ixsystems.com/community/threads/step-by-step-to-install-openvpn-inside-a-jail-in-freenas-11-1-u1.61681/](https://www.ixsystems.com/community/threads/step-by-step-to-install-openvpn-inside-a-jail-in-freenas-11-1-u1.61681/)
* [https://www.howtogeek.com/427982/how-to-encrypt-and-decrypt-files-with-gpg-on-linux/](https://www.howtogeek.com/427982/how-to-encrypt-and-decrypt-files-with-gpg-on-linux/)
* [https://forums.freebsd.org/threads/gnupg-in-a-jail-agent_genkey-failed-end-of-file-error.55182/] (https://forums.freebsd.org/threads/gnupg-in-a-jail-agent_genkey-failed-end-of-file-error.55182/)
* [https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=205246](https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=205246)
* [https://www.ixsystems.com/community/threads/openvpn-issues-in-new-jails-after-11-1.59828/](https://www.ixsystems.com/community/threads/openvpn-issues-in-new-jails-after-11-1.59828/)
* [https://github.com/kylemanna/docker-openvpn/issues/339](https://github.com/kylemanna/docker-openvpn/issues/339)
* [https://forums.gentoo.org/viewtopic-t-1086424-start-0.html](https://forums.gentoo.org/viewtopic-t-1086424-start-0.html)
* [https://wiki.ubuntu.com/nobody](https://wiki.ubuntu.com/nobody)
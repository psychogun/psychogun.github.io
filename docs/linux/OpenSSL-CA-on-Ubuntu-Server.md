---
layout: default
title: OpenSSL Certificate Authority on Ubuntu Server
parent: Linux
nav_order: 12
---
# OpenSSL Certificate Authority on Ubuntu Server
{: .no_toc }
This is how I configured a CA server on Ubuntu. My main source of information is copied from [https://www.digitalocean.com/community/tutorials/how-to-set-up-and-configure-a-certificate-authority-ca-on-ubuntu-20-04](Digital Ocean). Check them out.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}
---
## Getting started


## Prerequisites
* Proxmox 6.1
* Ubuntu 20.04
* TLSv1.2

## Set your time zone
```bash
rivest@shamir-adleman:~$ date
Sat 11 Jan 21:22:53 GMT 2020
rivest@shamir-adleman:~$ sudo dpkg-reconfigure tzdata

Current default time zone: 'Europe/Paris'
Local time is now:      Sat Jan 11 22:24:07 CET 2020.
Universal Time is now:  Sat Jan 11 21:24:07 UTC 2020.

rivest@shamir-adleman:~$ $ date
Sat 11 Jan 22:24:16 CET 2020
```

## Disable IPv6
```bash
rivest@shamir-adleman:~$ nano /etc/default/grub
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
rivest@shamir-adleman:~$ sudo update grub
```

## Change NTP server
```bash
rivest@shamir-adleman:~$ nano /etc/systemd/timesyncd.conf 
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
rivest@shamir-adleman:~$ sudo systemctl restart systemd-timesyncd
```
## qemu-guest-agent
```bash
rivest@shamir-adleman:~$ sudo apt update
rivest@shamir-adleman:~$ sudo apt install qemu-guest-agent
```
Shut down the VM. Enable Qemu Guest Agent in Proxmox and start.

## Installing Easy-RSA
```bash
rivest@shamir-adleman:~$ sudo apt install easy-rsa
```
## Preparing a Public Key Infrastructure Directory
```bash
rivest@shamir-adleman:~$ mkdir ~/easy-rsa
rivest@shamir-adleman:~$ ln -s /usr/share/easy-rsa* ~/easy-rsa/
rivest@shamir-adleman:~$ chmod 700 /home/california/easy-rsa
rivest@shamir-adleman:~$ cd ~/easy-rsa
rivest@shamir-adleman:~/easy-rsa$ ./easyrsa init-pki
```
After completing this section you have a directory that contains all the files that are needed to create a Certificate Authority. In the next section you will create the private key and public certificate for your CA.

## Creating a Certificate Authority
```bash
rivest@shamir-adleman:~/easy-rsa$ nano vars
# Easy-RSA 3 parameter settings

# NOTE: If you installed Easy-RSA from your distro's package manager, don't edit
# this file in place -- instead, you should copy the entire easy-rsa directory
# to another location so future upgrades don't wipe out your changes.

# HOW TO USE THIS FILE
#
# vars.example contains built-in examples to Easy-RSA settings. You MUST name
# this file 'vars' if you want it to be used as a configuration file. If you do
# not, it WILL NOT be automatically read when you call easyrsa commands.
#
# It is not necessary to use this config file unless you wish to change
# operational defaults. These defaults should be fine for many uses without the
# need to copy and edit the 'vars' file.
#
# All of the editable settings are shown commented and start with the command
# 'set_var' -- this means any set_var command that is uncommented has been
# modified by the user. If you're happy with a default, there is no need to
# define the value to its default.

# NOTES FOR WINDOWS USERS
#
# Paths for Windows  *MUST* use forward slashes, or optionally double-esscaped
# backslashes (single forward slashes are recommended.) This means your path to
# the openssl binary might look like this:
# "C:/Program Files/OpenSSL-Win32/bin/openssl.exe"

# A little housekeeping: DON'T EDIT THIS SECTION
# 
# Easy-RSA 3.x doesn't source into the environment directly.
# Complain if a user tries to do this:
if [ -z "$EASYRSA_CALLER" ]; then
	echo "You appear to be sourcing an Easy-RSA 'vars' file." >&2
	echo "This is no longer necessary and is disallowed. See the section called" >&2
	echo "'How to use this file' near the top comments for more details." >&2
	return 1
fi

# DO YOUR EDITS BELOW THIS POINT

# This variable is used as the base location of configuration files needed by
# easyrsa.  More specific variables for specific files (e.g., EASYRSA_SSL_CONF)
# may override this default.
#
# The default value of this variable is the location of the easyrsa script
# itself, which is also where the configuration files are located in the
# easy-rsa tree.

#set_var EASYRSA	"${0%/*}"

# If your OpenSSL command is not in the system PATH, you will need to define the
# path to it here. Normally this means a full path to the executable, otherwise
# you could have left it undefined here and the shown default would be used.
#
# Windows users, remember to use paths with forward-slashes (or escaped
# back-slashes.) Windows users should declare the full path to the openssl
# binary here if it is not in their system PATH.

#set_var EASYRSA_OPENSSL	"openssl"
#
# This sample is in Windows syntax -- edit it for your path if not using PATH:
#set_var EASYRSA_OPENSSL	"C:/Program Files/OpenSSL-Win32/bin/openssl.exe"

# Edit this variable to point to your soon-to-be-created key directory.  By
# default, this will be "$PWD/pki" (i.e. the "pki" subdirectory of the
# directory you are currently in).
#
# WARNING: init-pki will do a rm -rf on this directory so make sure you define
# it correctly! (Interactive mode will prompt before acting.)

#set_var EASYRSA_PKI		"$PWD/pki"

# Define X509 DN mode.
# This is used to adjust what elements are included in the Subject field as the DN
# (this is the "Distinguished Name.")
# Note that in cn_only mode the Organizational fields further below aren't used.
#
# Choices are:
#   cn_only  - use just a CN value
#   org      - use the "traditional" Country/Province/City/Org/OU/email/CN format

set_var EASYRSA_DN	"org"

# Organizational fields (used with 'org' mode and ignored in 'cn_only' mode.)
# These are the default values for fields which will be placed in the
# certificate.  Don't leave any of these fields blank, although interactively
# you may omit any specific field by typing the "." symbol (not valid for
# email.)

set_var EASYRSA_REQ_COUNTRY	"FR"
set_var EASYRSA_REQ_PROVINCE	"Paris"
set_var EASYRSA_REQ_CITY	"Paris"
set_var EASYRSA_REQ_ORG  	"Lebulo"
set_var EASYRSA_REQ_EMAIL	"e-mail@gmail.com"
set_var EASYRSA_REQ_OUD		"Laboratoriè"

# Choose a size in bits for your keypairs. The recommended value is 2048.  Using
# 2048-bit keys is considered more than sufficient for many years into the
# future. Larger keysizes will slow down TLS negotiation and make key/DH param
# generation take much longer. Values up to 4096 should be accepted by most
# software. Only used when the crypto alg is rsa (see below.)

#set_var EASYRSA_KEY_SIZE	4096

# The default crypto mode is rsa; ec can enable elliptic curve support.
# Note that not all software supports ECC, so use care when enabling it.
# Choices for crypto alg are: (each in lower-case)
#  * rsa
#  * ec

set_var EASYRSA_ALGO		ec

# Define the named curve, used in ec mode only:

set_var EASYRSA_CURVE		secp384r1

# In how many days should the root CA key expire?

set_var EASYRSA_CA_EXPIRE	825

# In how many days should certificates expire?

set_var EASYRSA_CERT_EXPIRE	365

# How many days until the next CRL publish date?  Note that the CRL can still be
# parsed after this timeframe passes. It is only used for an expected next
# publication date.

# How many days before its expiration date a certificate is allowed to be
# renewed?
set_var EASYRSA_CERT_RENEW	30

set_var EASYRSA_CRL_DAYS	180

# Support deprecated "Netscape" extensions? (choices "yes" or "no".) The default
# is "no" to discourage use of deprecated extensions. If you require this
# feature to use with --ns-cert-type, set this to "yes" here. This support
# should be replaced with the more modern --remote-cert-tls feature.  If you do
# not use --ns-cert-type in your configs, it is safe (and recommended) to leave
# this defined to "no".  When set to "yes", server-signed certs get the
# nsCertType=server attribute, and also get any NS_COMMENT defined below in the
# nsComment field.

#set_var EASYRSA_NS_SUPPORT	"no"

# When NS_SUPPORT is set to "yes", this field is added as the nsComment field.
# Set this blank to omit it. With NS_SUPPORT set to "no" this field is ignored.

#set_var EASYRSA_NS_COMMENT	"Easy-RSA Generated Certificate"

# A temp file used to stage cert extensions during signing. The default should
# be fine for most users; however, some users might want an alternative under a
# RAM-based FS, such as /dev/shm or /tmp on some systems.

#set_var EASYRSA_TEMP_FILE	"$EASYRSA_PKI/extensions.temp"

# !!
# NOTE: ADVANCED OPTIONS BELOW THIS POINT
# PLAY WITH THEM AT YOUR OWN RISK
# !!

# Broken shell command aliases: If you have a largely broken shell that is
# missing any of these POSIX-required commands used by Easy-RSA, you will need
# to define an alias to the proper path for the command.  The symptom will be
# some form of a 'command not found' error from your shell. This means your
# shell is BROKEN, but you can hack around it here if you really need. These
# shown values are not defaults: it is up to you to know what you're doing if
# you touch these.
#
#alias awk="/alt/bin/awk"
#alias cat="/alt/bin/cat"

# X509 extensions directory:
# If you want to customize the X509 extensions used, set the directory to look
# for extensions here. Each cert type you sign must have a matching filename,
# and an optional file named 'COMMON' is included first when present. Note that
# when undefined here, default behaviour is to look in $EASYRSA_PKI first, then
# fallback to $EASYRSA for the 'x509-types' dir.  You may override this
# detection with an explicit dir here.
#
#set_var EASYRSA_EXT_DIR	"$EASYRSA/x509-types"

# OpenSSL config file:
# If you need to use a specific openssl config file, you can reference it here.
# Normally this file is auto-detected from a file named openssl-easyrsa.cnf from the
# EASYRSA_PKI or EASYRSA dir (in that order.) NOTE that this file is Easy-RSA
# specific and you cannot just use a standard config file, so this is an
# advanced feature.

#set_var EASYRSA_SSL_CONF	"$EASYRSA/openssl-easyrsa.cnf"

# Default CN:
# This is best left alone. Interactively you will set this manually, and BATCH
# callers are expected to set this themselves.

#set_var EASYRSA_REQ_CN		"ChangeMe"

# Cryptographic digest to use.
# Do not change this default unless you understand the security implications.
# Valid choices include: md5, sha1, sha256, sha224, sha384, sha512

#set_var EASYRSA_DIGEST		"sha512"

# Batch mode. Leave this disabled unless you intend to call Easy-RSA explicitly
# in batch mode without any user input, confirmation on dangerous operations,
# or most output. Setting this to any non-blank string enables batch mode.

#set_var EASYRSA_BATCH		""
```

To create the root public and private key pair for your Certificate Authority, run the `./easy-rsa` command again, this time with the `build-ca` option:
```bash
rivest@shamir-adleman:~/easy-rsa$ ./easyrsa build-ca
```
In the output, you’ll see some lines about the OpenSSL version and you will be prompted to enter a passphrase for your key pair. Be sure to choose a strong passphrase, and note it down somewhere safe. You will need to input the passphrase any time that you need to interact with your CA, for example to sign or revoke a certificate.

You will also be asked to confirm the Common Name (CN) for your CA. The CN is the name used to refer to this machine in the context of the Certificate Authority. You can enter any string of characters for the CA’s Common Name. I suggest making the Common Name something that you’ll recognize as your root certificate in a list of other certificates. That’s really the only thing that matters.

You now have two important files — `~/easy-rsa/pki/ca.crt` and `~/easy-rsa/pki/private/ca.key` — which make up the public and private components of a Certificate Authority.

`ca.crt` is the CA’s public certificate file (root certificate). Users, servers, and clients will use this certificate to verify that they are part of the same web of trust. Every user and server that uses your CA will need to have a copy of this file. All parties will rely on the public certificate to ensure that someone is not impersonating a system and performing a Man-in-the-middle attack.

`ca.key` is the private key that the CA uses to sign certificates for servers and clients. If an attacker gains access to your CA and, in turn, your `ca.key` file, you will need to destroy your CA. This is why your `ca.key` file should only be on your CA machine and that, ideally, your CA machine should be kept offline when not signing certificate requests as an extra security measure.

With that, your CA is in place and it is ready to be used to sign certificate requests, and to revoke certificates.

## Distributing your Certificate Authority’s Public Certificate

Now your CA is configured and ready to act as a root of trust for any systems that you want to configure to use it. You can add the CA’s certificate to your OpenVPN servers, web servers, mail servers, and so on. Any user or server that needs to verify the identity of another user or server in your network should have a copy of the `ca.crt` file imported into their operating system’s certificate store.

To import the CA’s public certificate into a second Linux system like another server or a local computer, first obtain a copy of the `ca.crt` file from your CA server. You can use the `cat` command to output it in a terminal, and then copy and paste it into a file on the second computer that is importing the certificate. You can also use tools like `scp`, `rsync` to transfer the file between systems. However we’ll use copy and paste with `nano` in this step since it will work on all systems.

As your non-root user on the CA Server, run the following command:

```bash
rivest@shamir-adleman:~/easy-rsa$ cat ~/easy-rsa/pki/ca.crt
```
There will be output in your terminal that is similar to the following:

```bash
-----BEGIN CERTIFICATE-----
MIIDSzCCAjOgAwIBAgIUcR9Crsv3FBEujrPZnZnU4nSb5TMwDQYJKoZIhvcNAQEL
BQAwFjEUMBIGA1UEAwwLRWFzeS1SU0EgQ0EwHhcNMjAwMzE4MDMxNjI2WhcNMzAw
. . .
. . .
-----END CERTIFICATE-----
```
Copy everything, including the `-----BEGIN CERTIFICATE-----` and `-----END CERTIFICATE-----` lines and the dashes.

On your second Linux system use `nano` or your preferred text editor to open a file called `/tmp/ca.crt`:

```bash
ohio@michigan:~$ nano /tmp/ca.crt
```
Paste the contents that you just copied from the CA Server into the editor. When you are finished, save and close the file. If you are using `nano`, you can do so by pressing CTRL+X, then Y and ENTER to confirm.

Now that you have a copy of the `ca.crt` file on your second (Linux) system, it is time to import the certificate into its operating system certificate store.

### MacOS
* Open the macOS Keychain app
* Go to File > Import Items…
* Select your private key file (i.e. `ca.crt`)
* Search for whatever you answered as the Common Name name above

* Double click on your root certificate in the list
* Expand the Trust section
* Change the When using this certificate: select box to “Always Trust”

* Close the certificate window
* It will ask you to enter your password (or scan your finger), do that
* 🎉 Celebrate!

### FreeBSD
On FreeBSD, run the following commands (??):

### Ubuntu & Debian
On Ubuntu and Debian based systems, run the following commands as your non-root user to import the certificate:
```bash
ohio@michigan:~$ sudo cp /tmp/ca.crt /usr/local/share/ca-certificates/
ohio@michigan:~$ sudo update-ca-certificates
```

### RedHAT, CentOS, Fedora
To import the CA Server’s certificate on CentOS, Fedora, or RedHat based system, copy and paste the file contents onto the system just like in the previous example in a file called `/tmp/ca.crt`. Next, you’ll copy the certificate into `/etc/pki/ca-trust/source/anchors/`, then run the `update-ca-trust` command.

```bash
red@hat:~$ sudo cp /tmp/ca.crt /etc/pki/ca-trust/source/anchors/
red@hat:~$ sudo update-ca-trust
```
Now your second Linux system will trust any certificate that has been signed by the CA server.

### Firefox
If you are using your CA with web servers and use Firefox as a browser you will need to import the public `ca.crt` certificate into Firefox directly. Firefox does not use the local operating system’s certificate store. For details on how to add your CA’s certificate to Firefox please see this support article from [https://support.mozilla.org/en-US/kb/setting-certificate-authorities-firefox](Mozilla on Setting Up Certificate Authorities) (CAs) in Firefox.

### Windows
If you are using your CA to integrate with a Windows environment or desktop computers, please see the documentation on how to use `certutil.exe` to install a CA certificate.


## Creating and Signing a Certificate Request
Now that you have a Certificate Authority ready to use, you can practice generating a private key and certificate request to get familiar with the signing and distribution process.

A Certificate Signing Request (CSR) consists of three parts: a public key, identifying information about the requesting system, and a signature of the request itself, which is created using the requesting party’s private key. The private key will be kept secret, and will be used to encrypt information that anyone with the signed public certificate can then decrypt.

The first step that you need to complete to create a CSR is generating a private key. To create a private key using `openssl`, create a `ssl` directory and then generate a key inside it. We will make this request for a fictional server called `Deluge`, as opposed to creating a certificate that is used to identify a user or another CA.

### Private key
Let us create the private key for a service:
```bash
root@Deluge:~ # cd /home/deluge/.config/
root@Deluge:/home/deluge/.config # mkdir ssl
root@Deluge:/home/deluge/.config # cd ssl/
```
#### RSA
If you would like to create a private key for the `Deluge` system, using `RSA` and a length of 4096 keys;
```bash
root@Deluge:/home/deluge/ssl # openssl genrsa -out Deluge.key 4096
```

#### EC
However; I opted to use elliptic curves:
```bash
root@Deluge:/home/deluge/.config/deluge/ssl # openssl ecparam -name secp384r1 -out secp384r1.pem
root@Deluge:/home/deluge/.config/deluge/ssl # openssl ecparam -in secp384r1.pem -genkey -noout -out Deluge.key
root@Deluge:/home/deluge/.config/deluge/ssl # more Deluge.key 
-----BEGIN EC PRIVATE KEY-----
MIGkAgEBBDDDSXlrR/5wd6g4hXm6qSsGidwgcU7NLBD+xamjBVSqj/jZ2V7Nl7hH
kGGv3zsZItCgawYFK4EEACKhZANiAATAlsY+e/zG744zW3wbLCAyLMMa3GMDbyzG
VXHkhl5uLPj8Hcg5wuxd7YsnGtZw9Wb3AhrXo3z1BYcsqdRYdASn78ZcVHb+ermN
dlKDjQDVLx8yt5RdMPaiDCojJrZTFfM=
-----END EC PRIVATE KEY-----
```
### CSR / request file
Now that you have a private key you can create a corresponding CSR, again using the `openssl` utility. You will be prompted to fill out a number of fields like Country, State, and City. You can enter a `.` if you’d like to leave a field blank, but be aware that if this were a real CSR, it is best to use the correct values for your location and organization:

```bash
root@Deluge:/home/deluge/.config/ssl # openssl req -new -key Deluge.key -out Deluge.req
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:FR
State or Province Name (full name) [Some-State]:Paris
Locality Name (eg, city) []:Paris
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Lebulo
Organizational Unit Name (eg, section) []:Laboratoriè
Common Name (e.g. server FQDN or YOUR name) []:192.168.14.44
Email Address []:e-mail@gmail.com

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:

```
Please take a note that I am using the IP-address of the server for the Common Name. 

Copy the `Deluge.req` file to your CA server using `scp`:
```bash
root@Deluge:/home/deluge/.config/ssl # scp Deluge.req rivest@shamir-adleman:/tmp/Deluge.req
```

In this step you generated a Certificate Signing Request for a fictional server called `Deluge`. In a real-world scenario, the request could be from something like a staging or development web server that needs a TLS certificate for testing; or it could come from an OpenVPN server that is requesting a certificate so that users can connect to a VPN. In the next step, we’ll proceed to signing the certificate signing request using the CA Server’s private key.

## Signing a CSR
In the previous step, you created a practice certificate request and key for a fictional server. You copied it to the `/tmp` directory on your CA server (`shamir-adleman`), emulating the process that you would use if you had real clients or servers sending you CSR requests that need to be signed.

Continuing with the fictional scenario, now the CA Server needs to import the practice certificate and sign it. Once a certificate request is validated by the CA and relayed back to a server, clients that trust the Certificate Authority will also be able to trust the newly issued certificate.

~~Since we will be operating inside the CA’s PKI where the `easy-rsa` utility is available, the signing steps will use the `easy-rsa` utility to make things easier, as opposed to using the openssl directly like we did in the previous example.~~

~~The first step to sign the fictional CSR is to import the certificate request using the `easy-rsa` script:~~
```bash
rivest@shamir-adleman:~/easy-rsa$ ./easyrsa import-req /tmp/Deluge.req Deluge
```
### Subject Alternate Naming (SAN)
Create a Subject Alternate Naming (SAN) config file, specific for our fictional service Deluge:
```bash
rivest@shamir-adleman:~/easy-rsa$ mkdir config
rivest@shamir-adleman:~/easy-rsa$ cd config/
rivest@shamir-adleman:~/easy-rsa/config$ nano Deluge

authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = deluge
DNS.2 = 192.168.14.44
```

Create a certificate (`config/Deluge.crt`) for the server Deluge. If you encrypted your CA key, you’ll be prompted for your password at this point:
```bash
rivest@shamir-adleman:~/easy-rsa$ openssl x509 -req -in /tmp/Deluge.req -CA pki/ca.crt -CAkey pki/private/ca.key -CAcreateserial -noout -out config/Deluge.crt -days 825 -sha512 -extfile config/Deluge
Signature ok
subject=C = FR, ST = Paris, L = Paris, O = Leburo, OU = Laboratoriè, CN = 192.168.14.44, emailAddress = e-mail@gmail.com
Getting CA Private Key
Enter pass phrase for pki/private/ca.key:
rivest@shamir-adleman:~/easy-rsa$ 
```

With those steps complete, you have signed the `Deluge.req` CSR using the CA Server’s private key in `/home/rivest/easy-rsa/pki/private/ca.key`. The resulting `Deluge.crt` file contains the practice server’s public encryption key, as well as a new signature from the CA Server. The point of the signature is to tell anyone who trusts the CA that they can also trust the `Deluge` certificate.

At this point, you would be able to use the issued certificate with something like a web server, a VPN, configuration management tool, database system, or for client authentication purposes.

For our purpose, copy the contents of `config/Deluge.crt` to `/home/deluge/.config/deluge/ssl/Deluge.crt`:
```bash
rivest@shamir-adleman:~/easy-rsa$ more config/Deluge.crt
root@Deluge:/home/deluge/.config/deluge/ssl # nano Deluge.crt
```
#### chown & chmod
```bash
root@Deluge:/home/deluge/.config/deluge/ssl # chown deluge Deluge.key 
root@Deluge:/home/deluge/.config/deluge/ssl # chown deluge Deluge.crt
root@Deluge:/home/deluge/.config/deluge/ssl # chmod 600 Deluge*
```
Edit Deluge's preferences and make it use those keys under HTTPS option (Interface).

Stop the jail and start the jail again.


## Mylar
Here is another service I opted to encrypt using my newly created CA, first, we create the private key:
```bash
root@Mylar:/usr/local/mylar3 # mkdir ssl
root@Mylar:/usr/local/mylar3 # chown mylar ssl
root@Mylar:/usr/local/mylar3 # cd ssl
root@Mylar:/usr/local/mylar3/ssl # openssl ecparam -name secp384r1 -out secp384r1.pem
root@Mylar:/usr/local/mylar3/ssl # openssl ecparam -in secp384r1.pem -genkey -noout -out Mylar.key
```
CSR:
```bash
root@Mylar:/usr/local/mylar3/ssl # openssl req -new -key Mylar.key -out Mylar.req
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:FR
State or Province Name (full name) [Some-State]:Paris
Locality Name (eg, city) []:Paris
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Lebulo
Organizational Unit Name (eg, section) []:Laboratoriè
Common Name (e.g. server FQDN or YOUR name) []:192.168.4.45
Email Address []:e-mail@gmail.com

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

Copy `Mylar.req`:
```bash
root@Mylar:/usr/local/mylar3/ssl # scp Mylar.req rivest@shamir-adleman:/tmp/Mylar.req
```
SAN:
```bash
rivest@shamir-adleman:~/easy-rsa/config$ nano Mylar
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = mylar
DNS.2 = 192.168.14.45
```

Sign and create `Mylar.crt`:
```bash
rivest@shamir-adleman:~/easy-rsa$ openssl x509 -req -in /tmp/Mylar.req -CA pki/ca.crt -CAkey pki/private/ca.key -CAcreateserial -out config/Mylar.crt -days 825 -sha512 -extfile config/Mylar
```

Copy contents of `config/Mylar.crt` to `/usr/local/mylar3/ssl/Mylar.crt`.


## DNS / hostname resolution
Pfsense > Services > DNS Forwarder > add host 

Deluge
localdomain
192.168.4.44

Mylar
localdomain
192.168.4.45



## Fault finding
### hash
In order for *service* to accept certificate, it should be used with the private key generated along with the CSR code submitted for the certificate activation.

You can check whether the certificate matches the private key using the following openssl commands:

```bash
openssl x509 -in /path/to/certificate.crt -noout -modulus | openssl sha1 
openssl rsa -in /path/to/private.key -noout -modulus | openssl sha1
```

They should be the same. 

### #How to verify CSR for SAN?
It will be a good idea to check if your CSR contains the SAN, which you specified above in an *san.cnf* file.
```bash
openssl req -noout -text -in sslcert.csr | grep DNS
```
Example:
```bash
[root@Chandan test]# openssl req -noout -text -in sslcert.csr | grep DNS
               DNS:bestflare.com, DNS:usefulread.com, DNS:chandank.com
[root@Chandan test]#
```

## Author
Mr. Johnson

## Acknowledgments
* [https://www.openssl.org/docs/man1.0.2/man1/x509.html](https://www.openssl.org/docs/man1.0.2/man1/x509.html)
* [https://wiki.openssl.org/index.php/TLS1.3](https://wiki.openssl.org/index.php/TLS1.3)
* [https://wiki.mozilla.org/Security/Server_Side_TLS](https://wiki.mozilla.org/Security/Server_Side_TLS)
* [https://malware.news/t/everyone-loves-curves-but-which-elliptic-curve-is-the-most-popular/17657](https://malware.news/t/everyone-loves-curves-but-which-elliptic-curve-is-the-most-popular/17657)
* [https://support.mozilla.org/en-US/kb/setting-certificate-authorities-firefox](https://support.mozilla.org/en-US/kb/setting-certificate-authorities-firefox)
* [https://support.apple.com/en-us/HT210176](https://support.apple.com/en-us/HT210176)
* [https://deliciousbrains.com/ssl-certificate-authority-for-local-https-development/](https://deliciousbrains.com/ssl-certificate-authority-for-local-https-development/)
* [https://www.digitalocean.com/community/tutorials/openssl-essentials-working-with-ssl-certificates-private-keys-and-csrs](https://www.digitalocean.com/community/tutorials/openssl-essentials-working-with-ssl-certificates-private-keys-and-csrs)
* [https://www.digitalocean.com/community/tutorials/how-to-set-up-and-configure-a-certificate-authority-ca-on-ubuntu-20-04](https://www.digitalocean.com/community/tutorials/how-to-set-up-and-configure-a-certificate-authority-ca-on-ubuntu-20-04)
* [https://linuxize.com/post/how-to-set-or-change-timezone-on-ubuntu-20-04/](https://linuxize.com/post/how-to-set-or-change-timezone-on-ubuntu-20-04/)
* [https://blog.lbdg.me/freebsd-adding-self-signed-certificate-authority/](https://blog.lbdg.me/freebsd-adding-self-signed-certificate-authority/)
* [https://blog.socruel.nu/freebsd/how-to-install-private-CA-on-freebsd.html](https://blog.socruel.nu/freebsd/how-to-install-private-CA-on-freebsd.html)
* [https://wiki.openssl.org/index.php/Command_Line_Elliptic_Curve_Operations](https://wiki.openssl.org/index.php/Command_Line_Elliptic_Curve_Operations)
---
layout: default
title: How to configure Samba in an iocage jail on FreeNAS
parent: FreeNAS
nav_order: 14
---

# How to configure Samba in an iocage jail on FreeNAS
{: .no_toc }
I wanted to isolate a jail with VLAN from my network and use Samba to share some folders on this network. 

## Table of contents
{: .no_toc .text-delta }
1. TOC
{:toc}
---
## Getting started
### Prerequisites
* FreeNAS
* SMB Shares
* Different users & groups

## Create a new jail
Open up your FreeNAS gui > Jails > ADD.

Select an appropriate jail name (Samba)

# Configuration of mount points
Back to our FreeNAS gui. 

## Add group
Add a group in FreeNAS web gui. Call it "Documents_ro" (a group which is read-only ). Take note of the `GID` (1050).

## Edit ACL
Edit the dataset you want to share through samba, e.g. `/mnt/Tank/Documents` with mount point `/mnt/Documents` and add the `Documents_ro` group with read permission on the dataset.

## Add Mount points
Stop the jail and add mount points through FreeNAS web gui. I have mounted `/mnt/Tank/Documents` to mount point `/mnt/Documents`. Make the mount point read-only. Now, lets start the jail. 

## Configure of the jail
Log in to your FreeNAS through SSH, e.g. 
```bash
poco:~ loco$ ssh -l root 172.16.58.71
root@172.16.58.71's password: 
root@freenas:~ # 
```
List available jails.
```bash
root@freenas:~ #  iocage list
+-----+----------------+-------+--------------+------+
| JID |      NAME      | STATE |   RELEASE    | IP4  |
+=====+================+=======+==============+======+
| 11  | Samba          | up    | 11.3-RELEASE | DHCP |
+-----+----------------+-------+--------------+------+
```
Log in to your actual jail and update && upgrade:
```bash
root@freenas:~ # iocage console Samba
root@Samba:~ # pkg update && pkg upgrade
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
Checking for upgrades (1 candidates): 100%
Processing candidates (1 candidates): 100%
Checking integrity... done (0 conflicting)
Your packages are up to date.
root@Samba:~ # 
```

## Branch: latest
Let us update all our packages from `quarterly` to `latest`:
```bash
root@Samba:/mnt # cd /etc/pkg/
root@Samba:/etc/pkg # mkdir -p /usr/local/etc/pkg/repos
root@Samba:/etc/pkg # printf 'FreeBSD: { \n  url: "pkg+http://pkg.FreeBSD.org/${ABI}/latest", \n  mirror_type: "srv", \n  signature_type: "fingerprints", \n  fingerprints: "/usr/share/keys/pkg", \n  enabled: yes \n}' > /usr/local/etc/pkg/repos/FreeBSD.conf
```
Then do another `pkg update && pkg upgrade`.


# Create a group
Create a group in the jail with the same `GID` as the group from the FreeNAS gui above.
```bash
root@Samba:/mnt # pw groupadd Documents_ro -g 1050
```

## Add a service user
Add a user which will act as a enabled user to log in to our Samba shares. This user is called `smbuser` with `uid=8675309`, has `/nonexistent` home directory and sets the user's login shell to `/usr/sbin/nologin` which denies this user interactive login- and a comment is also provided to this user, `-c`.
```bash
root@Samba:/etc/pkg # pw adduser smbuser -u 8675309 -d /nonexistent -s /usr/sbin/nologin -c "Samba user"
```

Which groups does this user belong to?
```bash
root@Samba:/usr/local # id mylar
uid=8675309(smbuser) gid=8675309(smbuser) groups=8675309(smbuser)
```

Add the user `smbuser` to our `Documents_ro group:
```bash
root@Samba:/usr/local # pw usermod smbuser -G Documents_ro
root@Samba:/usr/local # id smbuser
uid=8675309(smbuser) gid=8675309(smbuser) groups=8675309(smbuser),1050(Documents_ro)
```

# Create a user
Add a user in the jail:
```bash
root@Samba:/mnt # adduser
Username: user1
Full name: User Number 1
Uid (Leave empty for default): 
Login group [user1]: 
Login group is user1. Invite max into other groups? []: Documents_ro
Login class [default]: 
Shell (sh csh tcsh nologin) [sh]: nologin
Home directory [/home/user1]: 
Home directory permissions (Leave empty for default): 
Use password-based authentication? [yes]: 
Use an empty password? (yes/no) [no]: no
Use a random password? (yes/no) [no]: yes
Lock out the account after creation? [no]: no
Username   : user1
Password   : <random>
Full Name  : User Number 1
Uid        : 1001
Class      : 
Groups     : user1 Documents_ro
Home       : /home/user1
Home Mode  : 
Shell      : /usr/sbin/nologin
Locked     : no
OK? (yes/no): y
adduser: INFO: Successfully added (user1) to the user database.
adduser: INFO: Password for (user1) is: sgHjPKfI3a
Add another user? (yes/no): no
Goodbye!
```

```bash
root@Samba:/mnt # id user1
uid=1001(user1) gid=1001(user1) groups=1001(user1),1050(Documents_ro)
```

Check to see if the user `user1` has the right permissions on the mounted ACL dataset:
```bash
root@Samba:/mnt # getfacl Documents/
# file: Documents/
# owner: 1000
# group: 1002
            group@:rwxpDdaARWc---:fd-----:allow
group:Documents_ro:r-x---a-R-c---:fd-----:allow
```
If you are able to see `group:Documents_ro`, instead of `group:1050`, all is good. 

All good. ZrZapTvg

# VLAN
To enable VLAN `20`on your iocage jail, add this to `rc.conf` (`epair0b` is the main interface, you can find out yours by using `ifconfig`):
```bash
root@Samba:/mnt # pkg install nano
root@Samba:/mnt # nano /etc/rc.conf
# VLAN20_Samba
vlans_epair0b="20"
ifconfig_epair0b_20="DHCP"
```

Start `dhclient` on the interface:
```bash
root@Samba:/mnt # service netif restart
root@Samba:/mnt # dhclient epair0b.20
DHCPDISCOVER on epair0b.20 to 255.255.255.255 port 67 interval 3
DHCPOFFER from 192.168.20.1
DHCPREQUEST on epair0b.20 to 255.255.255.255 port 67
DHCPACK from 192.168.20.1
bound to 192.168.20.3 -- renewal in 3600 seconds.
root@Samba:/mnt # 
```

# Samba
```bash
root@Samba:/etc/pkg # pkg search samba
p5-Samba-LDAP-0.05_2           Manage a Samba PDC with an LDAP Backend
p5-Samba-SIDhelper-0.0.0_3     Create SIDs based on G/UIDs
samba-nsupdate-9.14.2_1        nsupdate utility with GSS-TSIG support
samba410-4.10.13               Free SMB/CIFS and AD/DC server and client for Unix
root@Samba:/etc/pkg # pkg install samba410

(...)
Message from samba410-4.10.13:

--
How to start: http://wiki.samba.org/index.php/Samba4/HOWTO

* Your configuration is: /usr/local/etc/smb4.conf

* All the relevant databases are under: /var/db/samba4

* All the logs are under: /var/log/samba4

* Provisioning script is: /usr/local/bin/samba-tool

For additional documentation check: http://wiki.samba.org/index.php/Samba4
```

## Configure Samba
Create a `smb4.conf` file under `/usr/local/etc/`:
```bash
root@Samba:/etc/pkg # nano /usr/local/etc/smb4.conf     

[global]

# workgroup = NT-Domain-Name or Workgroup-Name, eg: WORKGROUP
   workgroup = WORKGROUP

# server string is the equivalent of the NT Description field
   server string = FreeNAS jail Samba Server

# Security mode. Defines in which mode Samba will operate. Possible
# values are share, user, server, domain and ads. Most people will want
# user level security. See the Samba-HOWTO-Collection for details.
   security = user
   map to guest = Bad User

# This option is important for security. It allows you to restrict
# connections to machines which are on your local network. The
# following example restricts access to two C class networks and
# the "loopback" interface. For more examples of the syntax see
# the smb.conf man page
   hosts allow = 14.

# Uncomment this if you want a guest account, you must add this to /etc/passwd
# otherwise the user "nobody" is used
;  guest account = pcguest

# this tells Samba to use a separate log file for each machine
# that connects
   log file = /var/log/samba4/log.%m

# Put a capping on the size of the log files (in Kb).
   max log size = 500

# log level 
   log level = 3

# Backend to store user information in. New installations should
# use either tdbsam or ldapsam. smbpasswd is available for backwards
# compatibility. tdbsam requires no further configuration.
   passdb backend = tdbsam

# Configure Samba to use multiple interfaces
# If you have multiple network interfaces then you must list them
# here. See the man page for details.
   interfaces = 192.168.20.3/24

# Browser Control Options:
# set local master to no if you don't want Samba to become a master
# browser on your network. Otherwise the normal election rules apply
   local master = no

# Disable broadcast binding in nmbd, by adding this Samba setting
    nmbd bind explicit broadcast = no 

[Documents]
    comment = Documents
    path = /mnt/Documents
    public = no
    browsable = yes
    writable = yes
    printable = no
    guest ok = no
    valid users = smbuser
# Enable the vfs_zfsacl module
# https://fossies.org/linux/samba/docs/manpages/vfs_zfsacl.8
   vfs objects = zfsacl 
# Configuration of vfs_zfsacl module
    nfs4:mode = special
    nfs4:acedup = merge
    nfs4:chown = no
```

Enable the `samba_server` to start at boot:
```bash
root@Samba:/etc/pkg # echo 'samba_server_enable="YES"' >> /etc/rc.conf
```

Start `samba_server`:
```bash
root@Samba:/etc/pkg # service samba_server start
```

### Add a Samba user
FreeBSD user accounts must be mapped to the SambaSAMAccount database for Windows® clients to access the share. Map existing FreeBSD user accounts using `pdbedit`:
```bash
root@Samba:/mnt # pdbedit -a smbuser
new password:
retype new password:
Forcing Primary Group to 'Domain Users' for smbuser
Forcing Primary Group to 'Domain Users' for smbuser
Forcing Primary Group to 'Domain Users' for smbuser
Forcing Primary Group to 'Domain Users' for smbuser
Unix username:        smbuser
NT username:          
Account Flags:        [U          ]
User SID:             
Primary Group SID:    
Full Name:            Smbuser Samba user
Home Directory:       \\Samba\smbuser
HomeDir Drive:        
Logon Script:         
Profile Path:         \\Samba\smbuser\profile
Domain:               
Account desc:         
Workstations:         
Munged dial:          
Logon time:           0
Logoff time:          9223372036854775807 seconds since the Epoch
Kickoff time:         9223372036854775807 seconds since the Epoch
Password last set:    Sun, 29 Mar 2020 01:59:11 CET
Password can change:  Sun, 29 Mar 2020 01:59:11 CET
Password must change: never
Last bad password   : 0
Bad password count  : 0
Logon hours         : FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
root@Samba:/mnt # 
```

We can try our share locally:
```bash
root@Samba:~ # smbclient '\\192.168.20.3\Documents' --user=smbuser
Enter WORKGROUP\smbuser's password: 
Try "help" to get a list of possible commands.
smb: \> ls
  Important documents      D        0  Sat Dec 29 22:54:12 2018
  Must read                D        0  Wed Mar 20 14:34:54 2019
smb: \> 
```

## Fault finding
### smbclient -L   
List shares that are available at ip:
```bash
root@Samba:~ # smbclient -L 192.168.101.3
Enter MYGROUP\root's password: 
Anonymous login successful

	Sharename       Type      Comment
	---------       ----      -------
	Documents       Disk      Documents
	IPC$            IPC       IPC Service (Samba Server Version 4.10.13)
Reconnecting with SMB1 for workgroup listing.
Anonymous login successful

	Server               Comment
	---------            -------

	Workgroup            Master
	---------            -------
	MYGROUP              
```

### smbstatus
```bash
root@Samba:~ # smbstatus

Samba version 4.10.13
PID     Username     Group        Machine                                   Protocol Version  Encryption           Signing              
----------------------------------------------------------------------------------------------------------------------------------------

Service      pid     Machine       Connected at                     Encryption   Signing     
---------------------------------------------------------------------------------------------

No locked files
```

## Authors
Mr. Johnson

## Acknowledgments
* [https://gist.github.com/sdebnath/086874c5df8b68e0df69](https://gist.github.com/sdebnath/086874c5df8b68e0df69)
* [https://thnee.se/articles/samba-freebsd-zfs-nfsv4-acls/](https://thnee.se/articles/samba-freebsd-zfs-nfsv4-acls/)
* [https://serverfault.com/questions/757764/what-kind-of-acl-storage-to-use-for-a-samba-domain-controller-on-zfs-and-freebsd](https://serverfault.com/questions/757764/what-kind-of-acl-storage-to-use-for-a-samba-domain-controller-on-zfs-and-freebsd )
* [http://www.learnlinux.org.za/courses/build/net-admin/ch08s02.html](http://www.learnlinux.org.za/courses/build/net-admin/ch08s02.html)
* [https://forums.freebsd.org/threads/samba-4-x-missing-smb-conf-example.59658/](https://forums.freebsd.org/threads/samba-4-x-missing-smb-conf-example.59658/)
* [https://forums.freebsd.org/threads/samba-4-8-as-standalone-server.66231/](https://forums.freebsd.org/threads/samba-4-8-as-standalone-server.66231/   )
* [https://docs.oracle.com/cd/E19253-01/819-5461/gbchf/index.html](https://docs.oracle.com/cd/E19253-01/819-5461/gbchf/index.html)
* [https://forums.gentoo.org/viewtopic-t-1056514-start-0.html](https://forums.gentoo.org/viewtopic-t-1056514-start-0.html)
* [https://wiki.samba.org/index.php/Setting_up_a_Share_Using_Windows_ACLs](https://wiki.samba.org/index.php/Setting_up_a_Share_Using_Windows_ACLs)
* [https://forums.freebsd.org/threads/smb-problem-again.69346/](https://forums.freebsd.org/threads/smb-problem-again.69346/)
* [https://www.freebsd.org/doc/handbook/network-samba.html](https://www.freebsd.org/doc/handbook/network-samba.html)
* [https://www.thegeekdiary.com/how-to-add-or-delete-a-samba-user-under-linux/](https://www.thegeekdiary.com/how-to-add-or-delete-a-samba-user-under-linux/)
* [https://forums.freebsd.org/threads/how-to-restart-a-single-network-interface-properly.22494/](https://forums.freebsd.org/threads/how-to-restart-a-single-network-interface-properly.22494/)
* [https://www.davd.io/samba-fileserver-on-freebsd/](https://www.davd.io/samba-fileserver-on-freebsd/)
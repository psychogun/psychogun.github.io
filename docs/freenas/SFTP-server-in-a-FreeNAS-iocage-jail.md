---
layout: default
title: How to configure SSH to act as an SFTP server in an iocage jail on FreeNAS
parent: FreeNAS
nav_order: 13
---

# How to configure SSH to act as an SFTP server in an iocage jail on FreeNAS
{: .no_toc }
I wanted a secure way to share my datasets on FreeNAS. 

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

Select an appropriate jail name.

## Mount points
Add your different mount points and make sure to make them read only. 

## Configure of the jail
Log in to your FreeNAS through SSH, e.g. 
```bash
poco:~ loco$ ssh -l root 172.16.58.71
root@1172.16.58.71's password: 
root@freenas:~ # 
```
List available jails.
```bash
root@freenas:/mnt/Breaking/TV Shows # iocage list
+-----+----------------+-------+--------------+------+
| JID |      NAME      | STATE |   RELEASE    | IP4  |
+=====+================+=======+==============+======+
| 10  | SFTPman        | up    | 11.3-RELEASE | DHCP |
+-----+----------------+-------+--------------+------+
```
Log in to your actual jail and update && upgrade:
```bash
root@freenas:~ # iocage console SFTPman
root@SFTPman:~ # pkg update && pkg upgrade
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
root@SFTPman:~ # 
```
When inside of the jail, edit `/etc/rc.conf` and `# Enable SSH daemon 
sshd_enable="YES"`
```bash
root@SFTPman:~ # vi /etc/rc.conf
```
PS: Check out [http://www.atmos.albany.edu/daes/atmclasses/atm350/vi_cheat_sheet.pdf](http://www.atmos.albany.edu/daes/atmclasses/atm350/vi_cheat_sheet.pdf) if you are unsure how to use the `vi` editor. 

### Make a group called sftponly
Make a group called `sftponly`.
```bash
root@SFTPman:~ #  pw groupadd sftponly
```
### Edit /etc/ssh/sshd_config
Edit the `sshd_config` file and add these lines of code at the bottom:
```bash
root@SFTPman:~ # vi /etc/ssh/sshd_config 
# SFTP settings
Match Group sftponly
ChrootDirectory /mnt/
X11Forwarding no
AllowTcpForwarding no
ForceCommand internal-sftp
```
### Add a new user
Add a new user which is only used for SFTP access.
```bash
root@SFTPman:~ # adduser
Username: sftpuser1
Full name: SFTP User No 1
Uid (Leave empty for default): 1099
Login group [sftpuser1]: sftponly
Login group is sftponly. Invite sftpuser1 into other groups? []: 
Login class [default]: 
Shell (sh csh tcsh git-shell nologin) [sh]: nologin
Home directory [/home/sftpuser1]: 
Home directory permissions (Leave empty for default): 
Use password-based authentication? [yes]: 
Use an empty password? (yes/no) [no]: 
Use a random password? (yes/no) [no]: 
Enter password: 
Enter password again: 
Lock out the account after creation? [no]: 
Username   : sftpuser1
Password   : *****
Full Name  : SFTP User No 1
Uid        : 1099
Class      : 
Groups     : sftponly 
Home       : /home/sftpuser1
Home Mode  : 
Shell      : /bin/sh
Locked     : no
OK? (yes/no): yes
adduser: INFO: Successfully added (sftpuser1) to the user database.
Add another user? (yes/no): no
Goodbye!
```
## Add mount points
Back in FreeNAS gui, select your jail and add a mount point as read-only (if you do not want users to be able to write to the dataset, then). Or just have confidence in your permissions management :) Read on.. .

## Configure ssh
Find out what your ip address of the jail is.
```bash
root@SFTPman:~ # ifconfig
lo0: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> metric 0 mtu 16384
	options=600003<RXCSUM,TXCSUM,RXCSUM_IPV6,TXCSUM_IPV6>
	inet6 ::1 prefixlen 128 
	inet6 fe80::1%lo0 prefixlen 64 scopeid 0x1 
	inet 127.0.0.1 netmask 0xff000000 
	nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>
	groups: lo 
epair0b: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
	options=8<VLAN_MTU>
	ether ac:dd:20:2a:cc:08
	hwaddr 02:07:10:00:13:0b
	inet 172.16.58.76 netmask 0xffffff00 broadcast 172.16.58.112
	nd6 options=1<PERFORMNUD>
	media: Ethernet 10Gbase-T (10Gbase-T <full-duplex>)
	status: active
	groups: epair 
root@SFTPman:~ # 
```
Start the ssh daemon on the jail.
```bash
root@SFTPman:~ # /etc/rc.d/sshd start
```
Open up a new terminal window and verify that you are unable to log in with ssh from our user `sftpuser1`.
```bash
poco:~ loco$ ssh -l sftpuser 172.16.58.76
Password for sfpuser1@SFTPman:
This service allows sftp connections only.
```
### Try FileZilla
Check if you can log in with an SFTP application, I tried FileZilla.
There you'll see that you are able to log in, but you are unable to navigate in the file hierarchy. That is because the created user does not have any permissions to read folders which is mounted. Neither does the group of which the user belongs to (sftponly).

## getfacl
Navigate to `/mnt` and do an `ls -alh` to see the UID:GID.
```bash
root@SFTPman:~ # cd /mnt/
root@SFTPman:/mnt # ls -alh
total 306
drwxr-xr-x    4 root    wheel     4B Sep 12 13:45 .
drwxr-xr-x   17 root    wheel    22B Sep 12 13:41 ..
drwxrwx---+ 803 1000    1002    820B Sep 12 01:01 Files
root@SFTPman:~ # 
```
Here we see that the user which has UID 1000 is the owner, and the group owner is a group with GID 1002 of the folder called `Files`. Guest users does not have any access at all. 
But are there any other groups or users that have access to this mount point? 

Use `getfacl` to get the files/folders acces control lists.
```bash
root@SFTPman:/mnt # getfacl Files/
# file: Files/
# owner: 1000
# group: 1002
            group@:rwxpDdaARWc---:fd-----:allow
        group:1006:r-x---a-R-c---:fd-----:allow
        group:1050:r-x---a-R-c---:fd-----:allow
        group:1070:rwxp-daARWc---:fd-----:allow
          user:922:rwxpD-aARWc---:fd-----:allow
            owner@:rwxpDdaARWcCo-:fd-----:allow
root@Radarr:/mnt # 
```
What we are interested in, is to use one of the groups `group:1006` or `group:1050` which has read and execute access of the folder. 
Because this is only for accessing and reading the files in the `/mnt/Files/` folder.
If I would like to have the `sftpuser1` to be able to write to this mount point, I would have in the following section proceeded with creating a group on the jail with GID 922 or 1002 and added `sftpuser1` in that group.

### Create groups with specific GID
List the groups that our user `sftpuser1` currently belongs to.
```bash
root@SFTPman:/mnt # groups sftpuser1
sftponly
```
We want to make our user a member of the group which has GID 1006. 
But which group has the GID of 1002?

Check in FreeNAS gui > Accounts > Groups. In my case, the group `Filesgroup_ro` was the group with GID 1006. 

In our jail, we have to create the group with a GID of 1006.
```bash
root@SFTPman:/mnt # pw groupadd Filesgroup_ro -g 1006
```

Then put our user `sftpuser1` in the group which we want the user to belong to.
```bash
root@SFTPman:/mnt # pw usermod sftpuser1 -G sftponly,Filesgroup_ro
```
Now which groups does `sftpuser1` belong to?
```bash
root@SFTPman:/mnt # groups sftpuser1
sftponly Filesgroup_ro
```
Back in FileZilla, disconnect and reconnect- you'll se that the folders now are listed (right click and choose Refresh).

Now you just have to repeat for whatever folders you would like the users to connect to.

## Reading getfacl from Windows
As the owner of the mount point, which should be the same owner as listed in SMB services, map a network drive.

Drive: X
Folder: \\freenas\Files
Reconnect at logon (uncheck)
Connect using different credentials (check)

Username: username-as-in-owner
Password: password
Press OK.

Right-click on the drive and select Properties and then the fane Security.
Press Edit, choose "Add..". Under "Enter the object name to select", you write 'FREENAS\Filesgroup_ro' and click 'Check names'. Then this should fly right through, click OK.
Let the be a checkmark next to 'Read & execute', 'List folder contents' and 'Read'. Press Apply.

Then you'll have the group 'Filesgroup_ro' with sufficient permissions for our user `sftpuser` to download files via a secure SFTP connection. 

## Authors
Mr. Johnson

## Acknowledgments
* [http://bin63.com/how-to-set-up-an-sftp-user-on-freebsd](http://bin63.com/how-to-set-up-an-sftp-user-on-freebsd)
* [http://www.hardstaff.com/enabling-ssh-on-freebsd/](http://www.hardstaff.com/enabling-ssh-on-freebsd/)
* [https://www.cyberciti.biz/faq/freebsd-add-a-user-to-group/](https://www.cyberciti.biz/faq/freebsd-add-a-user-to-group/)
* [https://forums.freebsd.org/threads/modifying-user-id-and-group-id.52910/](https://forums.freebsd.org/threads/modifying-user-id-and-group-id.52910/)
* [https://docs.oracle.com/cd/E19253-01/819-5461/gbchf/index.html](https://docs.oracle.com/cd/E19253-01/819-5461/gbchf/index.html)
* [https://www.cyberciti.biz/faq/freebsd-add-a-user-to-group/](https://www.cyberciti.biz/faq/freebsd-add-a-user-to-group/)

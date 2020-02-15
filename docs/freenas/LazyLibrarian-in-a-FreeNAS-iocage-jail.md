---
layout: default
title: How to install LazyLibrarian in a FreeNAS iocage jail
parent: FreeNAS
nav_order: 7
---

# How to install LazyLibrarian
{: .no_toc }
Well, I wanted to organize all my magazines. 


## Table of contents
{: .no_toc .text-delta }
1. TOC
{:toc}
---
## Getting started


## Requirements
* FreeNAS 11.2

## Configuration of FreeNAS
### Group
Go to FreeNAS gui and add a new group.
FreeNAS > Accounts > Groups > ADD.
GID: 1070
Name: lazylib
(unchecked) Permit Sudo
(unchecked) Allow repeated GIDs

Click SAVE.

And we want to add a group which is used for readers with read only access.
GID: 1071
Name: lazylib_ro
(unchecked) Permit Sudo
(unchecked) Allow repeated GIDs

Click SAVE.

### User
Go to FreeNAS gui and add a new user.
FreeNAS > Accounts > Users > ADD >
Full Name: Lazy Librarian
Username: lazylib
Password: something random
Confirm password: something random

User ID: 1070
(unchecked) New Primary Group
Primary Group: lazylib

Home Directory: /nonexistent

Enable password login: No
Shell: nologin

Click SAVE. 

Add a Dataset called LazyLibrarian in your designated pool by going to FreeNAS gui > Storage > Pools. 

Name: LazyLibrarian
Comments: LazyLibrarian folder
Sync: Inherit (standard)
Compression level: Inherit (lz4)

Share Type
* Windows

Enable Atime: Inherit (on)
ZFS Deduplication: Inherit (off)
Case Sensitivity: Sensitive

Click SAVE.

Select the Dataset and click the three little dots to the right, and select `Edit permissions`.

ACL Type: Windows
v Apply User
User: lamb
v Apply Group
Group: lazylib

v Apply Permissions recursively

Make your main user as an axuilliary member of the group by going to FreeNAS gui > Accounts > Users > select your main user, in my case, the user is called `lamb` and click the three little dots to the right of `lamb` and select Edit. Make sure that the group `lazylib` is added under Auxilliary Groups. 

Click SAVE. 

### SMB Share
Go to FreeNAS gui > Sharing > Windows (SMB) Shares
Click ADD.

Select your newly created Dataset. 

Everything should be OK as default, click SAVE.

### Mount dataset
Go to FreeNAS gui > Jails and select LazyLibrarian. Stop it, if it has started.
Click the three little dots to the right of the jail, and click Mount points.

ACTIONS > +Add Mount Point.

Source: /mnt/Zulu/LazyLibrarian
Destination: /mnt/LazyLibrarian

(unchecked) Read-Only

Click SAVE. 

Now go to FreeNAS gui > Jails and start LazyLibrarian.

## Create iocage jail
Create a jail called LazyLibrarian from your FreeNAS gui.

## Installation of LazyLibrarian
Log in to your FreeNAS with SSH. 
```bash
eternal:~ lamb$ ssh -l root 192.168.50.213
root@192.168.50.213's password: 
```
List your jails.
```bash
root@freenas:~ # jls
   JID  IP Address      Hostname                      Path
    21                  LazyLibrarian                 /mnt/Zulu/iocage/jails/LazyLibrarian/root
```
Log in to you jail.
```bash
root@freenas:~ # iocage console LazyLibrarian
```

Check the FreeBSD version.
```bash
root@LazyLibrarian:~ # uname -ro
FreeBSD 11.2-STABLE
```
Ensure that your FreeBSD system is up to date.
```bash
root@LazyLibrarian:~ # pkg update && pkg upgrade -y
```
Install the necessary packages.
```bash
root@LazyLibrarian:~ # pkg install -y nano wget git screen py36-sqlite3 py36-openssl py36-urllib3 ghostscript9-agpl-base-9.27_2 bash
```

Back to your iocage jail, create a new user account with your preferred username, we will use lazylib. 
The one thing to make sure of, is that this users UID is the same as the user created in the FreeNAS GUI (1070).

PS: Setting the UID to 1070, will also create the GID of the login group `lazylib` to 1070. 
```bash
root@LazyLibrarian:~ # adduser
root@LazyLibrarian:~ # adduser
Username: lazylibrarian
Full name: ^C
root@LazyLibrarian:~ # adduser
Username: lazylib
Full name: Lazy Librarian
Uid (Leave empty for default): 1070
Login group [lazylib]: 
Login group is lazylib. Invite lazylib into other groups? []: 
Login class [default]: 
Shell (sh csh tcsh git-shell nologin) [sh]: nologin
Home directory [/home/lazylib]: 
Home directory permissions (Leave empty for default): 
Use password-based authentication? [yes]: yes
Use an empty password? (yes/no) [no]: no
Use a random password? (yes/no) [no]: yes
Lock out the account after creation? [no]: yes
Username   : lazylib
Password   : <random>
Full Name  : Lazy Librarian
Uid        : 1070
Class      : 
Groups     : lazylib 
Home       : /home/lazylib
Home Mode  : 
Shell      : /usr/sbin/nologin
Locked     : yes
OK? (yes/no): 
OK? (yes/no): yes
adduser: INFO: Successfully added (lazylib) to the user database.
adduser: INFO: Password for (lazylib) is: Rree34PQc
adduser: INFO: Account (lazylib) is locked.
Add another user? (yes/no): no
Goodbye!
```
Change directory into newly created users home folder.
```bash
root@LazyLibrarian:~ # cd /home/lazylib/
```

Git download the repo.
```bash
root@LazyLibrarian:/home/lazylib # git clone https://gitlab.com/LazyLibrarian/LazyLibrarian.git
Cloning into 'LazyLibrarian'...
remote: Enumerating objects: 25365, done.
remote: Counting objects: 100% (25365/25365), done.
remote: Compressing objects: 100% (6967/6967), done.
remote: Total 25365 (delta 18340), reused 25141 (delta 18161)
Receiving objects: 100% (25365/25365), 17.28 MiB | 2.49 MiB/s, done.
Resolving deltas: 100% (18340/18340), done.
root@LazyLibrarian:/home/lazylib # 
```
Change permission of the folder.
```bash
root@LazyLibrarian:/home/lazylib # ls -alh
drwxr-xr-x  12 root     lazylib    26B Nov 18 10:13 LazyLibrarian
root@LazyLibrarian:/home/lazylib # chown -R lazylib LazyLibrarian/
root@LazyLibrarian:/home/lazylib # ls -alh
drwxr-xr-x  12 lazylib  lazylib    26B Nov 18 10:13 LazyLibrarian
```

Start LazyLibrarian as the user `lazylib` in a new `screen` session.
```bash
root@LazyLibrarian:/home/lazylib # screen
root@LazyLibrarian:/home/lazylib # pw unlock lazylib
root@LazyLibrarian:/home/lazylib # cd LazyLibrarian
root@LazyLibrarian:/home/lazylib/LazyLibrarian # su -m lazylib -c 'python3.6 LazyLibrarian.py'
```
If there are no errors and you are happy, you could launch LazyLibrarian in daemon mode.
```bash
root@LazyLibrarian:/home/lazylib/LazyLibrarian # su -m lazylib -c 'python3.6 LazyLibrarian.py -d'
```

(Ctrl + a, then d to detach from screen).

Check that LazyLibrarian is running as user `lazylib`.
```bash
root@LazyLibrarian:~ # top
last pid: 64954;  load averages:  2.43,  2.63,  2.84                                                                                                                                                                  up 67+15:46:37  10:35:39
12 processes:  1 running, 11 sleeping
CPU:  8.5% user,  0.1% nice,  7.6% system,  0.1% interrupt, 83.7% idle
Mem: 3579M Active, 47G Inact, 21G Laundry, 20G Wired, 2100M Free
ARC: 12G Total, 4713M MFU, 3955M MRU, 11M Anon, 123M Header, 3047M Other
     4352M Compressed, 6145M Uncompressed, 1.41:1 Ratio
Swap: 10G Total, 1130M Used, 9109M Free, 11% Inuse

  PID USERNAME    THR PRI NICE   SIZE    RES STATE   C   TIME    WCPU COMMAND
62448 lazylib      13  20    0   101M 57136K select 15   0:01   0.01% python3.6
```

Log in to you LazyLibrian by going to http://ip-adress:5299.


## Configuration of LazyLibrarian
Create a folder named `Magazines` inside of `/mnt/LazyLibrarian`.
```bash
root@LazyLibrarian:/mnt/LazyLibrarian # su -m lazylib -c 'mkdir Magazines'
```
Go to LazyLibrarian web gui  > Config > Processing

Magazine Foldername Pattern:
/mnt/LazyLibrarian/Magazines/$Title/$IssueDate

(unchecked) Magazines inside book folder

Magazine Filename Pattern: 
$Title - $IssueDate 

(unchecked) Create cover files for magazines
(unchecked) Create opf files for magazines
v (checked) Rename existing magazines on libraryscan

Click Save Changes.

Go to LazyLibrarian web gui >  Importing > Issue Dates:
Click Save Changes.

$b $Y

$Y-$m-$d
Magazine

## Tips
### Move file into folder
Create a folder for each magazine based on the first part of the filename of the .pdf, and move the file in to the respective folder.

```bash
root@LazyLibrarian:/mnt/LazyLibrarian # nano script.sh
#!/usr/local/bin/bash
for f in *–*.pdf; do d=${f%% –*}; mkdir -p "$d" &&  mv "$f" "$d"; done

root@LazyLibrarian:/mnt/LazyLibrarian # chmod +x script.sh
```
Execute by placing the `script.sh` file inside unsorted magazine folder, then do `./script.sh` and let it work it's magic. 

Then move these folders into your Magazine folder, and hit Library Scan.

### Create a label in deluge
Open Deluge, Connection Manager and connect to your deluge daemon. 
Right-click on the Label pane to the left and press +Add Label, Name: lazylibrarian.

Click Preferences and go to YaRSS2, then select the RSS Feeds tab. 
Add Feed and enter an appropriate RSS Feed Url with the correct category. Click Save. 

Select the Subscriptions tab and Add Subscription.

Subscription name: Assorted Magazines
RSS Feed: The one you added above
Filter include (regex): assorted.magazines

and then go to options, and select Label lazylibrarian and click Save, Apply and then OK. 
Then open up a web browser to configure the deluge label.



### Is it a real hyphen?
Check if the - (above) is really a real hyphen.
```bash
root@LazyLibrarian:/mnt/LazyLibrarian # printf '–' | od -tx1 -An
          e2  80  93               
```
[http://www.ltg.ed.ac.uk/~richard/utf-8.cgi?input=–&mode=char](http://www.ltg.ed.ac.uk/~richard/utf-8.cgi?input=–&mode=char)

## Fault finding
### Unable to process ascii
That is an error I had during Library Scan which threw LazyLibrarian off. I thought it first had something to do with my files formatting, was it not UTF-8. But the locale was not set in my iocage jail, and thus the hyphen above could not be read properly. 

Make sure your `locale` in your iocage jail is set.
```bash
root@LazyLibrarian:/mnt/LazyLibrarian # locale
LANG=en_US.UTF-8
LC_CTYPE="en_US.UTF-8"
LC_COLLATE="en_US.UTF-8"
LC_TIME="en_US.UTF-8"
LC_NUMERIC="en_US.UTF-8"
LC_MONETARY="en_US.UTF-8"
LC_MESSAGES="en_US.UTF-8"
LC_ALL=
root@LazyLibrarian:/mnt/LazyLibrarian #
```

For more information on how to proceed, go here [https://www.b1c1l1.com/blog/2011/05/09/using-utf-8-unicode-on-freebsd/](https://www.b1c1l1.com/blog/2011/05/09/using-utf-8-unicode-on-freebsd/)

## Authors
Mr. Johnson

## Acknowledgments
[https://serverfault.com/questions/348482/how-to-remove-invalid-characters-from-filenames](https://serverfault.com/questions/348482/how-to-remove-invalid-characters-from-filenames)
[https://linux.die.net/man/1/convmv](https://linux.die.net/man/1/convmv)
[https://www.freebsd.org/doc/handbook/using-localization.html](https://www.freebsd.org/doc/handbook/using-localization.html)



---
layout: default
title: How to install ClamAV in a FreeNAS iocage jail
parent: FreeNAS
nav_order: 3
---

# How to install ClamAV in a FreeNAS iocage jail
{: .no_toc }
I wanted to have a antivirus scanner which scans my datasets. I opted for using ClamAV, and this is how I installed and configured it to run on my ioacge. 

## Table of contents
{: .no_toc .text-delta }
1. TOC
{:toc}
---

## Getting started

### nano
```bash
root@Tautulli:~ # pkg install nano
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
The following 3 package(s) will be affected (of 0 checked):

New packages to be INSTALLED:
	nano: 4.6
	indexinfo: 0.3.1
	gettext-runtime: 0.20.1

Number of packages to be installed: 3

The process will require 3 MiB more space.
686 KiB to be downloaded.

Proceed with this action? [y/N]: y
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

### pkg update && pkg upgrade
```bash
root@Tautulli:~ # pkg update && pkg upgrade
```


## Install ClamAV
```bash
root@ClamAV:~ # pkg install clamav
```

## Configure ClamAV
ClamAV with `clamscan` is doing a lot of work when it starts up and and on my system `clamscan` takes about 30 seconds to complete a scan on a single text file. The extra time is spent loading the virus database into memory and such, but those nice people from ClamAV have a ready-made way to avoid it. Use `clamdscan` instead.

Well, okay, it’s not quite that simple. The difference between these two tools is that plain `clamscan` loads its own virus database and does the processing itself whereas `clamdscan` is a thin client for the `clamd` daemon, which keeps its virus database in memory ready to use. So in order to use `clamdscan`, you need to have `clamd` running.

Instead of passing options as you would do with `clamscan`, a `clamd.conf` file is used instead. This is how I configured my `clamdscan.conf` file:

```bash
nano /usr/local/etc/clamd.conf

# Log time with each message.                             
# Default: no
LogTime yes

# Log additional information about the infected file, such as its
# size and hash, together with the virus name.
ExtendedDetectionInfo yes

Make clamd start at boot:

nano 
# Enable ClamAV clamd
clamav_clamd_enable="YES"

The rc. is here: 
root@ClamAV:/usr/local/etc/rc.d # ls
clamav-clamd		clamav-freshclam	clamav-milter


root@ClamAV:/usr/local/etc # service clamav-clamd start
Starting clamav_clamd.





To make the job easy, I have created two shell scripts which you need to save on your FreeNAS server.

This first script updates the Anti-Virus definitions database, performs a scan of the relevant shares and then prepares a log file for emailing, finally it removes any redundant log files.

You need to save the script `scanav.sh` within the jail- you can either store this directly within the Jail, or have it accessible via a Dataset which you have shared using Add Storage.

Please update the following two variables within the script with your email address, and the location of your shares (if mounted somewhere different than /mnt):
* `to_email="homer@simpson.com`
* `scanlocation="/mnt`

```bash
root@freenas:/mnt/Balls/FreeNAS/Sysadmin/scripts 
root@freenas:/mnt/Balls/FreeNAS/Sysadmin/scripts # iocage console ClamAV
root@ClamAV:~ # cd /mnt/Sysadmin/scripts/
root@ClamAV:/mnt/Sysadmin/scripts # more scanav.sh
#!/bin/sh

### Notes ###
## Shell scripts to update the ClamAV definations, then run a scan and prepare an email template ##
## This script is called from a master script running as a cron job on the FreeNAS server ##
## Master script is: run_clamav_scan.sh  ##
##
## Instructions: ##
## 1) To use this you need to create a Jail called "ClamAV" ##
## 2) Open a Shall to the jail and then run: "pkg update" ##
## 3) The run: "pkg install clamav" ##
## 4) You can then "exit" the Jail ##
## 5) Add the windows shares you wish to scan by using the Jail Add Storage feature ##
## 5a) Add the shares to same location you use in the variable: "scanlocation" ##
## 6) Setp a cronjob on the FreeNAS server to run a shell script on the FreeNAS server: "run_clamav_scan.sh" ##
## 7) The shell script "run_clamav_scan.sh" then connects to the Jail and runs this script. ##
## 8) Once finished, the "run_clamav_scan.sh" script emails a log to the email entered in the variable: "to_email" ##
##
## https://www.clamav.net/ ##
## ClamAV® is an open source (GPL) anti-virus engine used in a variety of situations including email scanning, web scanning, ##
## and end point security. It provides a number of utilities including a flexible and scalable multi-threaded daemon, a command ##
## line scanner and an advanced tool for automatic database updates. ##

### Parameters ###
## email address ##
to_email="homer@simpson.com"

## Top directory of the files/directories you wish to scan, i.e. the "Jail Add Storage" locations ##
scanlocation="/mnt"
### End ###

### Update anti-virus definations ###
freshclam -l /var/log/clamav/freshclam.log
### End ###

## Where and what to name the list containing modified files withing the last 9 days
list="/mnt/clamscan_list_mod9.txt"
## End

### Make a list containing modified files within the last 9 days
find "${scanlocation}" -type f -ctime -9 > "${list}"
## End

### Run the anti-virus scan ###
started=$(date "+ClamAV Scan started at: %Y-%m-%d %H:%M:%S")
clamscan -i --bytecode-timeout=180000 -l /var/log/clamav/clamscan.log --file-list="${list}"
finished=$(date "+ClamAV Scan finished at: %Y-%m-%d %H:%M:%S")
### End ###

### prepare the email ###
## Set email headers ##
(
    echo "To: ${to_email}"
    echo "Subject: ${started}"
    echo "MIME-Version: 1.0"
    echo "Content-Type: text/html"
    echo -e "\\r\\n"
) > /tmp/clamavemail.tmp

## Set email body ##
(
    echo "<pre style=\"font-size:14px\">"
    echo "${started}"
    echo ""
    echo "${finished}"
    echo ""
    echo "--------------------------------------"    
    echo "ClamAV Scan Summary"
    echo "--------------------------------------"
    tail -n 8 /var/log/clamav/clamscan.log
    echo ""
    echo ""    
    echo "--------------------------------------"
    echo "freshclam log file"
    echo "--------------------------------------"
    tail -n +2 /var/log/clamav/freshclam.log
    echo ""
    echo ""    
    echo "--------------------------------------"
    echo "clamav log file"
    echo "--------------------------------------"
    tail -n +4 /var/log/clamav/clamscan.log | sed -e :a -e '$d;N;2,10ba' -e 'P;D'
    echo "</pre>"
) >> /tmp/clamavemail.tmp

### Tidy Up ###
## Delete the freshclam log in preparation of a new log ##
rm /var/log/clamav/freshclam.log

## Delete the clamscan log in preparation of a new log ##
rm /var/log/clamav/clamscan.log
### End ###
root@freenas:/mnt/Kaneda/FreeNAS/Sysadmin/scripts # 
```

The second script is called `run_clamav_scan.sh`, and this scripts calls the script in Step 4, then sends the email and finally deletes the last log file.

This script needs to be accessible from the FreeNAS server, but can be stored within a Dataset, for example, on my server its store in 

/mnt/tank/freenas/Sysadmin/scripts/run_clamav_scan.sh

```bash
#!/bin/sh

### Execute a shall script on the ClamAV jail, which updates the Anti-Virus definations and then runs a scan ###

## Define the location where the "avscan.sh" shell script is located on the jail:
scriptlocation="/mnt/Sysadmin/scripts/"

## Execute the script ##
iocage exec ClamAV "$scriptlocation"scanav.sh

## email the log ##
sendmail -t < /mnt/tank/Jails/ClamAV/tmp/clamavemail.tmp

## Delete the log file ##
rm /mnt/tank/Jails/ClamAV/tmp/clamavemail.tmp
```

You need to create a cron job via the Tasks section of the FreeNAS UI which runs the script run_clamav_scan.sh, I suggest you schedule this to run at a time when the server is not in used, to stop any impact affecting users, as well as deciding how often you want it run, I have scheduled it for every 5 days.

Please Note:
If you get the following error message in the freshclam log, you can ignore it.  This is because you are not setting up ClamAV as a daemon (i.e. clamd service) in the Jail, and only using it as a on-demand scanner.

WARNING: Clamd was NOT notified: Can't connect to clamd through /var/run/clamav/clamd.sock: No such file or directory


Thats it!
When it completes you should receive an email stating the start/end times of the scan, a summary of results, and a list of files which ClamAV has found which could be suspicious.

I say "could be", as you can always get false-positives with anti-virus scanners, but at least this way you can make a decision on what to do with the file.

I hope this makes sense.  If you found this resource useful (or not) I would be grateful if you could feedback using the rating system!

Jonathan









===> Creating groups.
Creating group 'clamav' with gid '106'.
Using existing group 'mail'.
===> Creating users
Creating user 'clamav' with uid '106'.
Adding user 'clamav' to group 'mail'.
[ClamAV] [12/12] Extracting clamav-0.102.2,1: 100%

## Acknowledgments
https://doc.owncloud.org/server/9.1/admin_manual/configuration_server/antivirus_configuration.html
https://www.clamxav.com/BB/viewtopic.php?t=2022
http://lists.clamav.net/pipermail/clamav-users/2018-August/006772.html
https://lists.gt.net/clamav/users/2502
https://www.devguerrilla.com/notes/2014/09/linux-speeding-up-clamav-with-clamd-on-rhel/
https://forums.freebsd.org/threads/solved-clamav.45557/
http://www.xfiles.dk/guide-on-how-to-install-clamav-on-freebsd/
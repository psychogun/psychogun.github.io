---
layout: default
title: How to move a Dataset in FreeNAS
parent: FreeNAS
nav_order: 18
---

# How to move a Dataset in FreeNAS
{: .no_toc }
I started to get those pesky e-mails, saying that 'Space usage for pool "Detainer" is 98%. Optimal pool performance requires used space remain below 80%.'

So I wanted to move a Dataset, "Audio", to another Pool to free up some space. This is how I did it. 

## Table of contents
{: .no_toc .text-delta }
1. TOC
{:toc}
---
## Getting started
A pool containing the original dataset Audio named Detainer and a new pool named Destroyer.

## Requirements
* FreeNAS 11.2
* 2 pools 
* Understanding replication/snapshot tasks


## Periodic Snapshot Tasks
FreeNAS gui > Periodic Snapshot Tasks > ADD.
Pool/Dataset: Detainer/Audio
Recursive: v
Snapshot Lifetime: 2 Weeks
Begin: 00:00:00
End: 23:59:00
Interval: 5 minutes
Day of week: Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, Sunday
v Enabled

Press SAVE.

## Create a Dataset
FreeNAS gui > Storage > Pools > Destroyer and on the three little dots to the far right, click Add Dataset:
Name: Audio
Comments: Audio Library
Share Type: Windows

Press SAVE.

Select the newly created Dataset, and on the far right, click the three little dots and select Edit Permissions.
v Apply User
User: mogabe
v Apply Group
Group: audio

Apply Permissions recursively is not neseccary, as we do not have anything there yet. 

Press SAVE.

## Replication
FreeNAS gui > Replication Tasks > ADD.
Pool/Dataset: Detainer
Remote ZFS Pool/Dataset: Detainer
v Recursively Replicate Child Dataset Snapshots
v Delete Stale Snapshots on Remote System
Replication Stream Compression: lz4 (fastest)
Limit (kpbks): 0
Begin Time: 00:00:00
End Time: 23:59:00
v Enabled
Remote Hostname: localhost
Remote Port: 22
Encryption Cipher: standard
uncheck Dedicated User Enabled
Remote Hostkey* > SCAN SSH KEY

Click SAVE. 


Replicate the Dataset from the pool to another pool by using the Tasks > Replication Tasks. 
Pool: Detainer > Dataset 'Audio 
replicates to
Pool: Destroyer > Dataset 'Audio'

Wait until the replication is done. 

### Stop all services
We want to make sure that the jails/VMs you have that are using the original Dataset is shut down. This is to ensure file integrity, and that we are able to move all the _latest_ files.
For instance, I have the dataset Audio mounted in Plex as a read-only folder. 
I have it mounted with write permissions on the Audio dataset in Lidarr. 

#### Lidarr
FreeNAS gui > Jails > select Lidarr and press STOP.

#### Plex
FreeNAS gui > Jails > select Plex and press STOP. 

## Stop replication
FreeNAS gui > Tasks > Replication Tasks > click the three little dots on the right of our task, select Edit and disable it by removing the checkmark to the left of 'Enabled'.

## Remount
FreeNAS gui > Jails > Lidarr > click the three little dots on the right of Lidarr, select Mount Points. Delete the mount point by clicking the three dots and select Delete. Check 'Confirm' and press 'DELETE MOUNT POINT'.

ACTIONS > + Add Mount Point 
Source : /mnt/Destroyer/Audio
Destination : /mnt/Audio

Do the same for any other jails using the original Mount Point.

## Start all services
#### Plex
FreeNAS gui > Jails > select Plex and press START.
#### Lidarr
FreeNAS gui > Jails > select Lidarr and press START.

### Permissions
Select the three little dots on the far right of your original Dataset (Detainer/Audio) and click 'Edit permissions'. Take a note of what type of ACL it is using, who is the owner of the folder and which group is the owner of the folder.

And with a Windows computer, mount the share (if the dataset is used by SMB) and take a note of all the permissions under the Security tab under mount Properties.

## Sharing and Services
FreeNAS gui > Services > select the slider to the right of 'SMB' and stop the service. 

FreeNAS gui > Sharing > Windows (SMB) Shares and select Audio.

Edit the path so that it changed from 
/mnt/Detainer/Audio
to
/mnt/Destroyer/Audio

Click SAVE.

FreeNAS gui > Services > select the slider to the right of 'SMB' and start the service. 

## Delete Dataset
To finally free up space, delete the dataset. 
FreeNAS gui > Storage > Pools > select Detainer/Audio and click the three little dots to the right and press 'Delete Dataset'.

## Periodic Snapshot Tasks
Now we want to be sure to enable a periodic snapshot task to our new Audio dataset on the pool 'Destroyer'.

FreeNAS gui > Periodic Snapshot Tasks > ADD.
Pool/Dataset: Destroyer/Audio
Recursive: uncheck
Snapshot Lifetime: 2 Weeks
Begin: 00:00:00
End: 23:59:00
Interval: 2 hours
Day of week: Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, Sunday
v Enabled

Press SAVE.

## Replication Tasks
Edit our original replication task.



## Periodic Snapshot Tasks
After you have shut down all the services that might write to the original Dataset, go in FreeNAS gui > Tasks > Periodic Snapshot Tasks. 
Select and edit the Detainer/Audio task and alter the frequency down to 5 minutes. 

This is only necessary if you see that the size of 'Detainer\Audio' and 'Destroyer\Replicated\Audio' is not equal. 
You should now see under Replications Tasks that the task is changed from 'Up to date' to 'Sending'. 

Refresh that page until the Status has changed to 'Up to date'. 
Confirm by checking that the Used space is the same. 

## Authors
Mr. Johnson

## Acknowledgments




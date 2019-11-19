---
layout: default
title: How to install Radarr in a FreeNAS iocage jail
parent: FreeNAS
nav_order: 7
---

# How to install Radarr in a FreeNAS iocage jail
{: .no_toc }
This is how i installed a Radarr, an independent fork of Sonarr reworked for automatically downloading movies via Usenet and BitTorrent, on FreeNAS in a iocage jail.

## Table of contents
{: .no_toc .text-delta }
1. TOC
{:toc}
---

## Getting started
These instructions will tell you how to add storage to an iocage jail, download, install and run Radarr as a specific user, and integrate it with Deluge.

.. Or you could just install it by simply issue command `pkg install radarr` [https://github.com/Radarr/Radarr/wiki/Installation#freebsd-jail](https://github.com/Radarr/Radarr/wiki/Installation#freebsd-jail).

### Prerequisites
* Knowledge of SSH and how to navigate to your jail in FreeNAS
* FreeNAS 11.2 and knowledge of how to create a jail with shares and knowledge of UNIX folder and files permissions

## FreeNAS Webgui
Log in to your Freenas. Go to 
Update and upgrade your iocage jail first:
```tcsh
root@Airdccp:/ # pkg upgrade && pkg update
```
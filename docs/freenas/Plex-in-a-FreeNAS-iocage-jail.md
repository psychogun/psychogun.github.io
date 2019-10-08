---
layout: default
title: How to install Plex in a FreeNAS iocage jail
parent: FreeNAS
nav_order: 3
---

# How to install Plex in a FreeNAS iocage jail
{: .no_toc }
This is how i installed a client-server media player system and software suite a on FreeNAS in a standalone iocage jail.

## Table of contents
{: .no_toc .text-delta }
1. TOC
{:toc}
---

## Getting started

## Update plex
```bash
root@Plex:/ # service plexmediaserver_plexpass stop
root@Plex:/ # pkg update && pkg upgrade multimedia/plexmediaserver-plexpass
root@Plex:/ # service plexmediaserver_plexpass start
```



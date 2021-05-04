---
layout: default
title: Adding a cron job
parent: OPNsense
nav_order: 1
---
# Adding a cron job
{: .no_toc }
This is how I made sure my OPNsense firewall was synchronized by adding a cron job.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}
---
## Getting started
So, my goal is to have OPNsense synchronize time with NTS. And this is accomplished by using `Chrony`. 

I have disabled the default `Network Time` daemon completely, by, as of current version, stopping the `NTPD` service and remove the addresses which it syncs to (Peers).

Whatever happens on the LAN side of things, the clients will just use their default servers / and or DNS host overrides pointing to an interface which is serving time on the firwall (which is not NTS, but hey, who can you trust- if not your own firewall).

Anyways.

- [x] Chrony synchronizes NTP with NTS.

- [x] Chrony serves the LAN side which is manually configured in the "Allowed Networks section" (who do I have to ask if I want to have a shiny drop-down menu here?)

- [x] Clients on LAN gets time from the LAN address on port 123.

- [ ] The firewall itself.

Now, how do to get the firewall to synchronize, with itself? 

Well, by pointing to a Time Server which is local (in the Network Time section)? And then start `NTP` daemon.

But guess what, the port :123 is already in use by `Chrony`, and `NTPD` can not start.

---

## Prerequisites
* OPNsense 20.7.7_1-amd64
* Chrony

```
[test@opnsense ~]$ sudo ntpdate -v 192.168.1.1
10 Jan 03:16:06 ntpdate[56714]: ntpdate 4.2.8p12-a (1)
10 Jan 03:16:06 ntpdate[56714]: the NTP socket is in use, exiting
```

```bash
[test@opnsense ~]$ date
Sun Jan 10 03:20:22 CET 2021
[test@opnsense ~]$ sudo ntpdate -v -u 192.168.1.1
10 Jan 03:20:56 ntpdate[41007]: ntpdate 4.2.8p12-a (1)
10 Jan 00:15:26 ntpdate[41007]: step time server 192.168.1.1 offset -11136.467989 sec
[test@opnsense ~]$ date
Sun Jan 10 00:15:29 CET 2021
```
> -u Direct ntpdate to use an unprivileged port for outgoing packets.

This situation is currently solved by using a cron.
```bash
[test@opnsense /usr/local/opnsense/service/conf/actions.d]$ vi actions_ntpdate.conf 
[start]
command:/usr/local/sbin/ntpdate -u 192.168.1.1
parameters:
type:script
message:ntpdate -u 192.168.1.1
description:ntpdate -u 192.168.1.1
```

Restart `configd` to be able to test the script with `configctl` (and to make it become available in the drop-down menu under Cron jobs in the GUI):
```bash
[test@opnsense /usr/local/opnsense/service/conf/actions.d]$ sudo service configd restart
Stopping configd...done
Starting configd.
```

Test it:
```bash
[test@opnsense /usr/local/opnsense/service/conf/actions.d]$ sudo configctl ntpdate start
OK
```
---

## Authors
Mr. Johnson

---

## Acknowledgments
* [https://github.com/opnsense/core/issues/2012](https://github.com/opnsense/core/issues/2012)
* [https://github.com/opnsense/plugins/issues/2162](https://github.com/opnsense/plugins/issues/2162)
* [https://blog.cloudflare.com/secure-time/](https://blog.cloudflare.com/secure-time/)
* [https://fedoraproject.org/wiki/Changes/NetworkTimeSecurity](https://fedoraproject.org/wiki/Changes/NetworkTimeSecurity)
* [https://opensource.com/article/18/12/manage-ntp-chrony](https://opensource.com/article/18/12/manage-ntp-chrony)
* [https://www.thegeekdiary.com/centos-rhel-7-configuring-ntp-using-chrony/](https://www.thegeekdiary.com/centos-rhel-7-configuring-ntp-using-chrony/)




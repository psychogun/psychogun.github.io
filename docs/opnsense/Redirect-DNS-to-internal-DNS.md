---
layout: default
title: Redirect DNS to internal DNS
parent: OPNsense
nav_order: 3
---
# Redirect DNS to internal DNS
{: .no_toc }
This is how I made sure clients on the LAN could not use external DNS on port 53 by redirecting those requests to OPNsense internal DNS resolver (Unbound).

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Redirect DNS to internal DNS

OPNsense > Firewall > NAT and click + Add.

* Navigate to Firewall > NAT > Port Forward tab and click + Add
* Click fa-level-up Add to create a new rule
* Fill in the following fields on the port forward rule:
* Interface: LAN
* Protocol: TCP/UDP
* Destination: Invert Match checked, LAN Address
* Destination Port Range: DNS (53)
* Redirect Target IP: 127.0.0.1
* Redirect Target Port: DNS (53)
* Description: Redirect DNS
* NAT Reflection: Disable

---

## Fault finding

```bash
storj@server~$ dig microsoft.com | grep SERVER
;; SERVER: 127.0.0.53#53(127.0.0.53)
storj@server~$
```

```bash
storj@server~$ systemd-resolve --status | grep "DNS Server"
  Current DNS Server: 172.22.55.1
         DNS Servers: 172.22.55.1
```
---

## Authors
Mr. Johnson

---

## Acknowledgments
* [https://docs.netgate.com/pfsense/en/latest/recipes/dns-redirect.html](https://docs.netgate.com/pfsense/en/latest/recipes/dns-redirect.html)

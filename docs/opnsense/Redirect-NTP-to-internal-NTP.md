---
layout: default
title: Redirect NTP to internal NTP
parent: OPNsense
nav_order: 3
---
# Redirect NTP to internal NTP
{: .no_toc }
This is how I made sure clients on the LAN could not use external NTP servers on port 123, redirecting those requests to OPNsense internal NTP server (Chrony).

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}
---

## Redirect NTP to internal NTP

Navigate to Firewall > NAT > Port Forward tab and click + Add
Click fa-level-up Add to create a new rule
Fill in the following fields on the port forward rule:
Interface: LAN
Protocol: TCP/UDP
Destination: Invert Match checked, LAN Address
Destination Port Range: NTP (123)
Redirect Target IP: 127.0.0.1
Redirect Target Port: NTP (123)
Description: Redirect NTP
NAT Reflection: Disable

---

## Authors
Mr. Johnson

---

## Acknowledgments
* [https://docs.netgate.com/pfsense/en/latest/recipes/dns-redirect.html](https://docs.netgate.com/pfsense/en/latest/recipes/dns-redirect.html)

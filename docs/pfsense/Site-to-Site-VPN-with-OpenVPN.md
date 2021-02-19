---
layout: default
title: Site-to-Site VPN with OpenVPN
parent: pfSense
nav_order: 5
---

# Site-to-Site VPN with OpenVPN
{: .no_toc }
Here is how I configured two pfSense firewalls with site-to-site VPN.

<details open markdown="block">
  <summary>
   Table of contents
  </summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

{: .no_toc .text-delta }

---

## Getting started
Install the Suricata package by navigating to System > Package Manager > Available Packages.

### Prerequisites
* pfSense 2.4.4-RELEASE-p3 (amd64)

### Internet
* Local (Server): pfsense.texas.com
* Remote (Client): pfsense.houston.com

### Local (Server) pfsense.texas.com
* Local (VLAN101): 192.168.101.1/29
* Remote (VLAN100): 192.168.100.1/29

### Remote (Client) pfsense.houston.com
* Local (VLAN100): 192.168.100.1/29
* Remote (VLAN101): 192.168.101.1/29

### Tunnel network
* Tunnel network: 10.0.101.0/30

---

## Local (Server) configuration
### Certificate Authority
Go to System > Cert. Manager and create a Certifice Authority by clicking "+ Add".

img Site-to-Site-VPN-with-OpenVPN_01_Create-CA.png

Save it and export it (Export CA).

### Certificates
Go to System > Cert. Manager and go to the Certificates tab. Create a new certificate 

img Site-to-Site-VPN-with-OpenVPN_02_Add-Certificate.png

img Site-to-Site-VPN-with-OpenVPN_03_Certificate-Attributes.png

Save it and Export Certificate and Export Key.

### OpenVPN
Select VPN > OpenVPN and click + Add. 

img Site-to-Site-VPN-with-OpenVPN_04_General-Information

img Site-to-Site-VPN-with-OpenVPN_05_Cryptographic-Settings.png

img Site-to-Site-VPN-with-OpenVPN_06_Cryptographic-Settings-02.png

img Site-to-Site-VPN-with-OpenVPN_07_Cryptographic-Settings-03.png


### Interfaces Assignment
Add the OpenVPN connection to an interface. 

---

## Authors
Mr. Johnson

---

## Acknowledgments
* [https://docs.netgate.com/pfsense/en/latest/vpn/openvpn/configuring-a-site-to-site-pki-ssl-openvpn-instance.html](https://docs.netgate.com/pfsense/en/latest/vpn/openvpn/configuring-a-site-to-site-pki-ssl-openvpn-instance.html)
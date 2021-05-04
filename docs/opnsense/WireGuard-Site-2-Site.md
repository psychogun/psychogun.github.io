---
layout: default
title: WireGuard Site-2-Site
parent: OPNsense
nav_order: 5
---

# WireGuard Site-2-Site
{: .no_toc }
This is how I could use a computer on my own network (<kbd>Site B</kbd>) and use services which were on a remote site (<kbd>Site A</kbd>) as if they were on my local area network. 

Clients on Site A could access CCTV resources on Site B. 

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
With the use of OPNsense firewall at <kbd>Site A</kbd> (where the services / resources are) and pfSense at <kbd>Site B</kbd> (where the clients were), plus WireGuard installed on both firewalls, I enabled a fully encrypted tunnel in no time. 

Communication between these sites are encrypted when travelling through the internet by WireGuard. I did more granular security utilizing firewall rules at both sites, on both WG interfaces. 

I suggest taking a minute or two to read the Conceptual Overview on the official WireGuard website:
* [https://www.wireguard.com](https://www.wireguard.com)


---

## Prerequisites
### Site A 
* OPNsense 20.7.8_4-amd64
Public IP: 123.44.55.66.77 (obviously not mine)
WireGuard Internal IP: 10.0.88.1
VLAN 22: 172.10.22.0/24 (used for network management)
VLAN 50: 172.10.50.0/24 (server resources)
VLAN 77: 10.0.77.0/24 (clients on Site A)

### Site B
* pfSense 2.5.0-RELEASE (amd64) 
Public IP: 88.99.77.66 (obviously not mine)
WireGuard Internal IP: 10.0.88.2
VLAN 5: 192.168.5.0/24 (full access on every LAN @ Site A)
VLAN 200: 192.168.200.0/24 (access only to server resources on Site A)
VLAN 40: 10.0.40.0/24 (CCTV network)


---

Let's begin.
## Site A - WireGuard 
### Local
`Local` can be thought of as "Server".

Hit the `+` icon.

* Enabled: [x]
* Name: SiteA
* Instance:
* Public Key: leave blank (will be generated after `Save`)
* Private Key: leave blank (will be generated after `Save`)
* Listen Port: 51821 (change this to your liking)
* DNS Server: 
* Tunnel Address: 10.0.88.1/24 (this will be the address of the WG0 interface on this specific firewall)
* Peers: leave blank for now (we will come back to this section)
* Disable Routes: leave unchecked

Hit `Save`. We will come back to this configuration page in a moment. 

### Endpoints
Hit `+` icon

* Name: SiteBsite2site
* Public Key: leave blank (will be added after)
* Shared Secret: leave blank (will be added after)
* Allowed IPs: 10.0.88.2/32, 192.168.5.0/24, 192.168.200.0/24
* Endpoint Address: Public IP of Site B
* Endpoint Port: 51825 (remember this port; you'll do portforwarding later on in your pfSense firewall)
* KeepAlive: 25

Hit `Save`. We will come back to this endpoint configuration page in a moment.

PS: PS: 
* Allowed IPs: First, you have to allow the remote wg0 interface to traverse the tunnel (10.0.88.2).
Secondly, the next IP(s) you define here, is the local network on your remote site from which you want to allow traffic to your network on. Each peer / client / endpoint (a client) will be able to send packets to the network interface with a source IP matching his corresponding list of allowed IPs. For example, when a packet is received by the server from peer / Site B, after being decrypted and authenticated, if its source IP is 192.168.200.0/24 then it's allowed onto the interface; otherwise it's dropped.

This is however, call it, just basic security. We'll do more granular security in a bit, in the Firewall Rules section of the particular `wg` interface. 

By default, WireGuard tries to be as silent as possible when not being used; it is not a chatty protocol. For the most part, it only transmits data when a peer wishes to send packets. When it's not being asked to send packets, it stops sending packets until it is asked again. In the majority of configurations, this works well. However, when a peer is behind NAT or a firewall, it might wish to be able to receive incoming packets even when it is not sending any packets. Because NAT and stateful firewalls keep track of "connections", if a peer behind NAT or a firewall wishes to receive incoming packets, he must keep the NAT/firewall mapping valid, by periodically sending keepalive packets. This is called persistent keepalives. When this option is enabled, a keepalive packet is sent to the server endpoint once every interval seconds. A sensible interval that works with a wide variety of firewalls is 25 seconds. Setting it to 0 turns the feature off, which is the default, since most users will not need this, and it makes WireGuard slightly more chatty. This option will keep the "connection" open in the eyes of NAT.


Go back to the `Local` tab, open the instance and choose the newly created endpoint in `Peers`, Hit `Save`. 

### General
Enable wireguard. 

* Enable WireGuard: [x]
Click `Save`.

---

## Interfaces 
When you have enabled Wireguard in the section above, you'll notice that you will have a WireGuard tab in your Firewall > Rules section. This particular section is GLOBAL. Rules applied here, will apply to all WireGuard interfaces. Do not edit this one ruleset. at least, I am not editing that ruleset - leaving it to deny anything incoming. 

### Assignments
If everything went OK in the section above, you'll be able to assign a new interface called `wg0`. Assign this interface and enable it- use another name than "WireGuard" for this interface. Maybe call it <kbd>WG_Site_B</kbd>?

We will apply rules here in a bit. 

## Firewall 
### Rules
#### WAN
Enable incoming IPv4 UDP connections to 51821 on your WAN address.

#### WG_Site_B
You can add granular rules here for your incoming clients;
allow 192.168.5.0/24 to all your networks on Site A - like, for management?
allow 192.168.200.0/24 only to specific networks on Site A - like, for SMB traffic?


### NAT


---

## Site B - pfSense

## pfSense
### VPN
#### WireGuard
Click `+ Add Tunnel`.

* Description: SiteB
* Address: 10.0.88.2/24
* Listen Port: 51821
Click `Generate` to generate Interface Keys, then click `+ Add Peer`. 

* Description: Site A S2S
* Endpoint: Public IP of Site A
* Endpoint Port: 51821
* Keep Alive: 25
* Public Key: Copy in the `Local` server's Public Key (from OPNsense, `Local`)
* Allowed IPs: 
* Peer WireGuard Address: 10.0.88.1
Click Update.

Hit `Save`. 

PS: Public key: Hit the pencil (edit button) of your newly created server on Site A, `Local` - and copy your servers Public Key

Add a Passphrase here if you would like an additional layer of security (PS: Generate a PSK on your pfSense)

--- 

## OPNSense
Wireguard
## Endpoint


Copy the Public Key for this tunnel from pfSense. Go back to your Endpoint configuration in OPNsense and edit the connection. 
Paste in the public key. 


---

OPNsense as a server:

Go to tab Local and create a new instance. Give it a Name and set a desired Listen Port. If you have more than one service instance be aware that you can use the Listen Port only once. 

* Tunnel Address: 10.0.10.1/24
Hit **Save**.

Edit the connection again.

Write down the public key. 


## Endpoints
Name: client
Public Key: 

When this VPN is set up on OPNsense only do the same on the second machine and exchange the public keys. Now go to tab Endpoints and add the remote site, give it a Name, insert the Public Key and the Allowed IPs e.g. 192.168.0.2/32, 10.10.10.0/24. This will set the remote tunnel IP address (/32 is important when using multiple endpoints) and route 10.10.10.0/24 via the tunnel. Endpoint Address is the public IP of the remote site and you can also set optionally the Endpoint Port, now hit Save changes.

Go back to tab Local, open the instance and choose the newly created endpoint in Peers.

Now we can Enable the VPN in tab General and go on with the setup.





Tunnel Address: 10.0.88.1/24

---

## Fault finding
```bash
root@opnsense:/home/user1 # service wireguard restart
wg-quick: `wg0' is not a WireGuard interface
[#] wireguard-go wg0
INFO: (wg0) 2021/02/21 15:29:56 Starting wireguard-go version 0.0.20201118
[#] wg setconf wg0 /tmp/tmp.QpLpudt8/sh-np.n3cHZS
Unable to find port of endpoint: `siteb.asdf.net:'
Configuration parsing error
[#] rm -f /var/run/wireguard/wg0.sock
```

Endpoint address should be an IP - not FQDN (?).

---

## Authors
Mr. Johnson

---

## Acknowledgments
* [https://imgur.com/a/nVUscn8](https://imgur.com/a/nVUscn8)
* [https://homenetworkguy.com/how-to/configure-wireguard-opnsense/](https://homenetworkguy.com/how-to/configure-wireguard-opnsense/)
* [https://serversideup.net/how-to-configure-a-wireguard-macos-client/](https://serversideup.net/how-to-configure-a-wireguard-macos-client/)
* [https://rickfreyconsulting.com/wireguard-site-to-site-vpn-example/](https://rickfreyconsulting.com/wireguard-site-to-site-vpn-example/)
* [https://www.reddit.com/r/PFSENSE/comments/lmv1cp/how_to_setup_wireguard_on_pfsense_252102_with/](https://www.reddit.com/r/PFSENSE/comments/lmv1cp/how_to_setup_wireguard_on_pfsense_252102_with/)
* [https://docs.opnsense.org/manual/how-tos/wireguard-s2s.html](https://docs.opnsense.org/manual/how-tos/wireguard-s2s.html)
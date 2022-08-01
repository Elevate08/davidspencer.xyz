---
title: "WireGuard Setup on Pfsense"
date: 2021-11-13T10:43:26-05:00
tags: ['wireguard','pfsense','vpn']
categories: ['tutorial']
author: David Spencer
draft: false
---

In this post I will explain how to setup WireGuard on your pfsense router. Your
pfsense router will be the WireGuard server and I'll show a couple example client
configurations at the end.

---

## Installing WireGuard on pfsense

- pfsense version 2.5.2
- WireGuard version 0.1.5

Navigate to System > Package Manager > Available Packages

Search for WireGuard and Install.

---

## Configuring WireGuard Server

### Create Tunnel

Navigate to VPN > WireGuard

Create a tunnel by clicking Add Tunnel

- Disable Tunnel
- Add a Description
- Change the listen port or leave at default of 51820
- Generate New Keys
- Copy Public Key, you'll need it later when configuring a client

Click Save Tunnel

### Configure Interface and Enable Tunnel

Navigate to Interfaces > Assignments

On the Available Network Ports row, select the tun_wg0 interface and click add.

- Enable Interface
- Add a Description
- Static IPv4
- IPv6: None (unless preferred)
- IPv4 Address: Choose an IPv4 Address and Subnet Mask

Click Save

Navigate to VPN > WireGuard

Click Edit on your new tunnel and enable the tunnel

### Configure Firewall

Navigate to Firewall > Rules

Create a new WAN Rule

- Action: Pass
- Interface: WAN
- Address Family: IPv4
- Protocol: UDP
- Source: any
- Destination: WAN Address
- Destination Port Range: 51820
- Log Packets, if preferred
- Add a description
- Tag: vpn (if desired, not used but could be in other rules)

Click Save

Create a new VPN Rule (VPN is the name of the interface)

- Action: Pass
- Interface: VPN
- Address Family: IPv4
- Protocol: Any
- Source: any
- Destination: LAN net

Click Save

Repeat the VPN rules to allow all desired access

---

## Configuring a Client

I will use a CentOS Stream system for this example but any client can be used
the configuration is the same, installations will differ though.

### Install WireGuard

[WireGuard Installation](https://www.wireguard.com/install/)

For CentOS I followed Method 2

```
sudo yum install elrepo-release epel-release
sudo yum install kmod-wireguard wireguard-tools
```

### Key Generation

Once installed you can follow these instructions to generate a key [Key Generation](https://www.wireguard.com/quickstart/#key-generation)

```
umask 077
wg genkey | tee privatekey | wg pubkey > publickey
```

Your private key is stored in a file 'privatekey', use this in the next step
Your public key is stored in a file 'publickey', use this when adding the peer in pfsense later

### Interface Config File

```
vim /etc/wireguard/wg0.conf
[Interface]
PrivateKey = YLzz2QbAowMXx913RyXsr33FVlcNZ+wx96vVKtdL9nk= (privatekey generated in previous step)
Address = IP Address assigned to this client in VPN Subnet

[Peer]
PublicKey = iQSIKmjCbWR57bu0kk774Sg03t80dRyV64UXIv2ADFY= (public key generated during tunnel creation)
AllowedIPs = 192.168.0.0/24 (Any Subnets you want this client to route through the VPN Tunnel, can use 0.0.0.0/0 if you want all traffic routed through VPN)
EndPoint = 1.2.3.4:51820 (PFsense Router WAN IP Address or DNS Name and Wirguard Port configured on tunnel)
```

---

## Adding a Peer

Descriptions on the configurations below can be found [here](https://docs.netgate.com/pfsense/en/latest/vpn/wireguard/settings.html#wireguard-settings-peer)

Navigate to VPN > WireGuard > Peers

Add Peer

- Enable Peer
- Select the tunnel we created
- Add a description of the peer
- Public Key generated on client during configuring a client
- Allowed IPs should be the Interface Address from the client.

The Allowed IPs is what was a source of confusion for me when setting up the WireGuard Server.
I couldn't have more than one peer and it was due to setting the allowed IPs to 0.0.0.0/0 for both peers.
Remember the Allowed IPs are what traffic is routed through the tunnel, you can't have 2 default routes in this case.

Click Save Peer

---

## Testing Client

On your client run the following command

```
wg-quick up wg0
```

You should see some routes created

```
[#] ip link add wg0 type wireguard
[#] wg setconf wg0 /dev/fd/63
[#] ip -4 address add 10.10.10.10/32 dev wg0
[#] ip link set mtu 1420 up dev wg0
[#] ip -4 route add 0.0.0.0/0 dev wg0
```

You should now be able to test connectivity by pinging your WireGuard tunnel interface IP Address
This was set when the interface was assigned and you chose your WireGuard subnet [Configure Interface](#configure-interface)

To ensure the interface is configured at startup after any reboots use the following command

```
systemctl enable wg-quick@wg0
```

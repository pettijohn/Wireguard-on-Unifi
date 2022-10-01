# WireGuard-on-Unifi

This document is a brief supplementary tutorial to augment the [wireguard-vyatta-ubnt](https://github.com/WireGuard/wireguard-vyatta-ubnt/) project, with some additional conceptual introductions. I am a beginner at Wireguard, and this is how I understand it--YMMV, caveat emptor, etc. 

## Prerequisites

* Determine the public IP address of your EdgeRouter and, if required, set up dynamic DNS. 
* SSH enabled on EdgeRouter 
* WireGuard [installed](https://github.com/WireGuard/wireguard-vyatta-ubnt/wiki/EdgeOS-and-Unifi-Gateway) on EdgeRouter 

## Step-by-step 

This document assumes you have a mobile phone, a laptop, and a home network environment you want to grant them access to; in other words, a classic client-server VPN problem. 

### Define WireGuard IP Space

First, you need to define a *new private WireGuard IP space* for your WireGuard devices. This has no relation to the Internal IP space on your home network nor the public IP addresses--just invent it out of whole cloth. The only requirement is that it must not overlap with your home network's internal IP space. We'll connect it to your home network internal IP space later. Such as:

| Device | WireGuard IP |
|---|---|
| EdgeRouter| 192.168.33.1 |
| Phone | 192.168.33.2 |
| Laptop | 192.168.33.3 |

I'll call this the *WireGuard IP Space.* Again, this can be whatever you want, provided it is [private](https://en.wikipedia.org/wiki/Private_network#Private_IPv4_addresses). My EdgeRouter also has a *Public IP* from Comcast (73.x.x.x) and an *Internal IP* (192.168.0.1). 

### Generate Keypairs

Then you create keypairs on each device. The Android, iOS, and Windows apps do this in a GUI; copy the *public* key and send it to yourself. Keep the *private* key private, there's no need for it to leave the device. 

On your Edge Router, you'll use the `wg` command to generate a key. 

```
# Generate a private key and dump it to a file
wg genkey > /config/auth/wg.key 

# Generate a public key from the private - copy this
wg pubkey < /config/auth/wg.key
# and/or save it to a file
wg pubkey < /config/auth/wg.key > /config/auth/wg.public 
``` 

After generating the keys on all of your devices:

| Device | WireGuard IP | WG Public Key |
|---|---|---|
| EdgeRouter| 192.168.33.1 | DiRycQSVUfDLPPbCDh9IvwMCs8jq7suROWx6XLRhlGw= |
| Phone | 192.168.33.2 | VwNKbjC+OmMk383XcBcWTjsedsl5zrBUvm/DGY1MmzU= |
| Laptop | 192.168.33.3 | RBZCzKRnrKQPj+ybQ4PFDeQF1Ld8TXSvCgNl8Vb5x2M= |

### Bring up the WireGuard interface wg0

Now that you have the public key and Wirguard-IP of your EdgeRouter, time to connect the points. On the EdgeRouter, bring up a new WireGuard interface:

```
configure # enter unifi config editing mode 
set interfaces wireguard wg0 address 192.168.33.1/24
set interfaces wireguard wg0 listen-port 51820
set interfaces wireguard wg0 route-allowed-ips true
set interfaces wireguard wg0 private-key /config/auth/wg.key
```
This says to bring up WireGuard on a new interface, `wg0`. `wg0` is a network adapter, kind of like `eth0`, but even more like `lo` in that it's a virtual device. `wg0` listens on the new address we defined and the /24 subnet (not sure if /32 would also work), UDP port 51820, and the path to the *private* key. `wg0` listens on the "fake" WireGuard IP 192.168.33.1, but once we open up the firewall, it will also listen on the public internet; since it's *on* the EdgeRouter it also has access to the internal network. This is sort of mystifying - it exists in superposition on all networks. 

### Add peers 

Now that the interface is up, time to add some *peers,* who is allowed to connect and from what WireGuard IPs. One quirk of WireGuard is you need to allow the peer by public key *and* WireGuard IP. That's why we went to the trouble of defining the IP space up front. 

```
set interfaces wireguard wg0 peer VwNKbjC+OmMk383XcBcWTjsedsl5zrBUvm/DGY1MmzU= allowed-ips 192.168.33.2/32
set interfaces wireguard wg0 peer RBZCzKRnrKQPj+ybQ4PFDeQF1Ld8TXSvCgNl8Vb5x2M= allowed-ips 192.168.33.3/32
```

Now you've added the Phone and Laptop, both their public keys and their WireGuard IPs. Note we're using /32 mask. I think that peer IPs are mutually exclusive, and if there is any overlap (such as duplicating an IP or using /24) WireGuard will pick one winner to dedupe. You can also add a list of allowed IPs. 

Finally, open the firewall:

```
set firewall name WAN_LOCAL rule 20 action accept
set firewall name WAN_LOCAL rule 20 protocol udp
set firewall name WAN_LOCAL rule 20 description 'WireGuard'
set firewall name WAN_LOCAL rule 20 destination port 51820
```

WAN_LOCAL is the ruleset that allows Public Internet (WAN) traffic to access the EdgeRouter (local) device (where `wg0` is running). (As opposed to WAN_IN, which is the ruleset for port forwarding *in*to the internal network, *through* the EdgeRouter.) 

All done. Commit, save, and exit the `configure` mode of EdgeRouter.
```
commit
save
exit
```

Verify the config:
```
sudo wg show all
```

### Connect client device to server

On the Laptop device, create a new configuration.

```
[Interface]
PrivateKey = xxxxx=
Address = 192.168.33.3/24

[Peer]
PublicKey = DiRycQSVUfDLPPbCDh9IvwMCs8jq7suROWx6XLRhlGw=
AllowedIPs = 192.168.33.0/24, 192.168.0.1/24
Endpoint = vpn.example.com:51820
```

The *Interface* section is the private key generated by the app and the WireGuard IP we defined early on. The *Peer* section defines the server endpoint. Finally, after all these mystical IP addresses, we use a real IP address on the public Internet. Comcast in my case, 73.x.x.x, with dynamic DNS in front of it, and the port. Note that in AllowedIPs we route *two* address spaces, the `.33.0` space for WireGuard traffic, and also `.0.1` for the internal network behind the EdgeRouter. Because `wg0` on the EdgeRouter exists *locally and virtually and mystically* on the EdgeRouter, it has access to all networks at the same time, as long as you send traffic to it. If you want to route *all* traffic through the VPN, add `0.0.0.0/0` to AllowedIPs. 

That's it. Repeat on your phone or additional devices, then activate the VPN. 

## Further Reading

* https://www.erianna.com/wireguard-ubiquity-edgeos/
* https://blog.usman.network/posts/wireguard-vpn-on-a-ubiquiti-edgerouter/
* https://github.com/WireGuard/wireguard-vyatta-ubnt/wiki/EdgeOS-and-Unifi-Gateway

## Bonus - Dynamic DNS with Cloudflare

[This guide from Unifi](https://help.ui.com/hc/en-us/articles/204976324-EdgeRouter-Custom-Dynamic-DNS) didn't work for my, but [forum posts](https://community.ui.com/questions/SOLVED-Cloudflare-DDNS-1-9-0/fbe64c70-db03-4449-a046-3edc30aa9e8b?page=2) had a fix. Here's the full solution; substitute `eth0` with the WAN interface that has a public IP:

```
configure
set service dns dynamic interface eth0 service custom-cloudflare host-name <subdomain.domain.com>
set service dns dynamic interface eth0 service custom-cloudflare login <user@domain.com>
set service dns dynamic interface eth0 service custom-cloudflare password <cloudflare-api-key>
set service dns dynamic interface eth0 service custom-cloudflare protocol cloudflare
set service dns dynamic interface eth0 service custom-cloudflare options "zone=domain.com ssl=yes"
set service dns dynamic interface eth0 service custom-cloudflare server api.cloudflare.com/client/v4
commit ; save; exit
```

The last two are clutch - `ssl=yes` and *no trailing slash* on the server URL. And don't mess with the name - it needs to be `custom-cloudflare`.

Verify and/or update:

```
show dns dynamic status
update dns dynamic interface eth0
```
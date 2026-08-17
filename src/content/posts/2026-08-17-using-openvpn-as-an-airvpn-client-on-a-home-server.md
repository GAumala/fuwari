---
title: "Using OpenVPN as an AirVPN client on a home server"
published: 2026-08-17
description: "How I used OpenVPN as an AirVPN client on a Debian home server for private outbound traffic, inbound connections despite CGNAT, and a Tailscale exit-node setup."
tags: [OpenVPN, Self-hosted, Linux, Tailscale, Networking]
category: Self-hosted
---

I wanted my home server to use a VPN for outbound internet traffic. Not for anything exotic, just the boring privacy reason: when services on the server make connections to the public internet, I would rather have that traffic leave through a VPN provider instead of directly through my home connection.

I also had an inbound traffic problem. My ISP has me behind CGNAT, so my router does not have a normal public IP address I can forward ports from. That means services running at home cannot easily accept incoming connections from the public internet.

I went with [AirVPN](https://airvpn.org/) because it solves both sides of this setup:

- regular OpenVPN configuration files, so I do not have to run a custom client
- port forwarding, which lets selected services accept inbound connections even though my ISP has me behind CGNAT

It also supports Bitcoin payments. It's not a hard requirement for me, but nice to have. Really shows they are more privacy conscious than your average company.

The setup was simpler than I expected. The only extra wrinkle was that I also run Tailscale on the same server, and I wanted tailnet traffic to stay local instead of being pulled into the VPN tunnel.

:::note
My home server runs Debian 13 (Trixie). The following tutorial assumes Debian. If you are using a different distro additional steps may be required.
:::

## Install OpenVPN as a client

First, install OpenVPN on the server:

```shell
sudo apt update
sudo apt install openvpn
```

That gives you the OpenVPN client and the systemd service template used to start VPN profiles.

## Get an AirVPN config file

AirVPN lets you generate OpenVPN configuration files from the client area on their website. Pick the server or region you want and download the config.

For me, Miami had the best latency, so I generated a config for a server located there.

Copy the config into `/etc/openvpn/` and give it a clear name:

```shell
sudo cp ~/Downloads/airvpn-miami.ovpn /etc/openvpn/my-chosen-server.conf
```

The exact filename matters because systemd uses it as the service instance name. If the file is:

```text
/etc/openvpn/my-chosen-server.conf
```

then the service is:

```shell
sudo systemctl start openvpn@my-chosen-server
```

To enable it on boot:

```shell
sudo systemctl enable openvpn@my-chosen-server
```

That is the core setup. Once the service is running, the server has a full VPN tunnel. Outbound internet traffic goes through AirVPN, including traffic from Docker containers, while normal local network traffic still stays local.

You can check that the tunnel exists with:

```shell
ip addr
```

After OpenVPN connects, there should be a new interface named `tun0`.

## Keeping Tailscale traffic out of the VPN

The full tunnel is useful, but it creates a problem if the server also uses Tailscale.

Tailscale uses addresses in `100.64.0.0/10`. I use it to connect back to my home server remotely, so I do not want traffic for tailnet nodes to go through AirVPN. That traffic should go directly through my normal network route.

The fix is to add this route to the AirVPN OpenVPN config:

```shell
route 100.64.0.0 255.192.0.0 net_gateway
```

In my case that line went into:

```text
/etc/openvpn/my-chosen-server.conf
```

Then restart the VPN service:

```shell
sudo systemctl restart openvpn@my-chosen-server
```

What this does:

- `100.64.0.0 255.192.0.0` matches the Tailscale carrier-grade NAT range
- `net_gateway` tells OpenVPN to use the original network gateway instead of the VPN tunnel

So the routing behavior becomes:

- public internet traffic goes through AirVPN
- LAN traffic stays local
- Tailscale traffic stays on the normal route too

That is exactly what I want on a home server. The VPN protects outbound internet traffic without breaking private remote access.

## Binding services to the VPN interface

Once OpenVPN is running, `tun0` gets its own VPN address. That gives you a useful option: bind a service to the VPN interface address so it only listens through the VPN path.

You can inspect the address with:

```shell
ip addr show tun0
```

The binding step depends on the service. Some services let you configure a bind address directly. Others need Docker networking or service-specific configuration. The important idea is the same: use the VPN interface address when you want that service to be reachable only through the VPN route.

## AirVPN port forwarding

If a service needs inbound connections from the public internet, binding to `tun0` is only half of the setup. You also need a forwarded port.

This matters because my home connection is behind CGNAT. CGNAT means the ISP is sharing one public IPv4 address across multiple customers. My router still has an internet connection, but it does not have its own publicly reachable IPv4 address, so normal router port forwarding is not enough.

With AirVPN, the public entry point is the forwarded port on AirVPN's side. Traffic reaches AirVPN first, then AirVPN sends it through the tunnel to my server.

AirVPN makes this straightforward:

1. Open the AirVPN client area.
2. Reserve a port.
3. Point the service at that port.
4. Configure the service to listen on the VPN interface.

After that, inbound connections to the reserved AirVPN port can reach the service through the VPN tunnel, even though the service is not reachable through the normal home internet connection.

This is the part that makes AirVPN a good fit for self-hosted services. Many VPN providers are fine for outbound privacy, but port forwarding is what makes the setup practical for software that expects inbound peer connections.

## Using AirVPN as a Tailscale exit node

Any device on my tailnet, like my phone, can now use my home server as an exit node, and automatically route all public internet traffic through AirVPN. This can be easily toggled ON/OFF with the Tailscale app. The server does need a little setup to advertise itself as an exit node on the tailnet. I just had to create `/etc/sysctl.d/99-tailscale.conf` with:

```
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
```

And then run:

```
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
sudo tailscale set --advertise-exit-node
sudo tailscale up
```

Be advised that it's also necessary to [allow the exit node from the admin console](https://tailscale.com/docs/features/exit-nodes#allow-the-exit-node-from-the-admin-console). Once that is ready, routing mobile traffic through AirVPN from devices on the tailnet is as simple as flipping a switch on the Tailscale app. Not that there is anything wrong with the [Eddie client](https://airvpn.org/android/eddie/) offered by AirVPN. If speed or latency matters, Eddie may be the better choice because the device connects to AirVPN directly, but for me it's more convenient to use a single VPN app because I already have Tailscale on multiple mobile devices to access my home server.

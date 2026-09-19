---
title: Traffic control for WireGuard
---

No—**the client’s traffic** going through the VPN does not mean that all traffic to the VPS stops going through Apache2 or Xray.

There are two separate traffic directions:

- **Inbound traffic to the VPS:** requests arriving at the VPS’s public IP and ports `80`, `443`, `5777`, or `7777` continue reaching the services that listen there.
- **VPN-client traffic:** packets entering the VPS through `wg0` are forwarded and masqueraded out through the VPS’s normal internet interface.

With this client configuration:

```ini
AllowedIPs = 0.0.0.0/0
```

only the **client’s traffic** is routed through the VPN. It does not globally redirect incoming VPS traffic or take over Apache2/Xray ports.

The server-side rules should be limited to WireGuard traffic:

```ini
PostUp = iptables -A FORWARD -i %i -j ACCEPT
PostUp = iptables -A FORWARD -o %i -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE

PostDown = iptables -D FORWARD -i %i -j ACCEPT
PostDown = iptables -D FORWARD -o %i -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
```

These rules affect packets whose source is the WireGuard subnet `10.8.0.0/24`. They do not redirect ports `80`, `443`, `5777`, or `7777`.

The risky configuration would be one that changes the VPS’s own default route or uses broad rules such as:

```bash
iptables -t nat -A OUTPUT ...
iptables -t nat -A PREROUTING ...
iptables -F
iptables -t nat -F
```

Those could affect server-generated traffic, port forwarding, or existing firewall rules. We should not use them.

Before and after enabling WireGuard, verify the listeners:

```bash
sudo ss -lntup | grep -E ':(80|443|5777|7777|51820)\b'
```

Also confirm that the default route remains through the normal interface:

```bash
ip route
```

Expected behavior:

```text
Internet client → VPS:443   → Apache2/Xray as currently configured
Internet client → VPS:5777  → WhatsApp Proxy as currently configured
Internet client → VPS:7777  → WhatsApp Proxy as currently configured
VPN client → wg0 → VPS forwarding/NAT → Internet
```

So the correct setup is a **VPN gateway for connected clients**, not a VPN redirector for every service running on the VPS.

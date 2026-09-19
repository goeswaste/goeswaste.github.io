---
title: Possible Conflicts with Other Services
---

Yes—WireGuard can coexist with Xary, Apache2, and WhatsApp Proxy without changing their listeners, provided we configure it carefully. I cannot honestly guarantee “in any possible way,” because firewall changes, routing changes, or an incorrect network-interface name could affect unrelated services.

The safe approach is:

- WireGuard listens only on **UDP 51820**.
- Apache2 continues listening on **TCP 80 and 443**.
- WhatsApp Proxy continues listening on its existing ports, **TCP/UDP 5777 and 7777 as applicable**.
- WireGuard uses its own interface, `wg0`, and private subnet such as `10.8.0.0/24`.
- We will not bind, proxy, redirect, or DNAT ports `80`, `443`, `5777`, or `7777`.
- We should not run `ufw enable` blindly, because an existing firewall policy could be changed and potentially block current services.
- The WireGuard NAT rules should be restricted to traffic entering through `wg0`, rather than broadly modifying traffic for all interfaces.

Before changing anything, create backups and record the current state:

```bash
sudo cp -a /etc/wireguard /etc/wireguard.backup.$(date +%F-%H%M%S) 2>/dev/null || true
sudo iptables-save | sudo tee /root/iptables-before-wireguard.rules >/dev/null
sudo ss -lntup | tee /root/listeners-before-wireguard.txt
sudo systemctl --no-pager --full status apache2
```

Verify the current services:

```bash
sudo ss -lntup | grep -E ':(80|443|5777|7777)\b'
```

Use this safer WireGuard server configuration pattern:

```ini
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY

PostUp = iptables -A FORWARD -i %i -j ACCEPT
PostUp = iptables -A FORWARD -o %i -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE

PostDown = iptables -D FORWARD -i %i -j ACCEPT
PostDown = iptables -D FORWARD -o %i -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE

[Peer]
PublicKey = CLIENT_PUBLIC_KEY
AllowedIPs = 10.8.0.2/32
```

Replace `eth0` with the actual public interface shown by:

```bash
ip route get 1.1.1.1
```

The `PostUp`/`PostDown` rules add forwarding and masquerading for the WireGuard subnet; they do not redirect ports or change Apache’s or the proxy’s listening sockets. This is the normal role of `wg-quick` hook rules. <citation src="1,3"></citation>

After starting WireGuard, immediately compare the state:

```bash
sudo systemctl enable --now wg-quick@wg0

sudo ss -lntup | grep -E ':(80|443|5777|7777|51820)\b'
sudo systemctl --no-pager --full status apache2
sudo wg show
```

You should see:

- Existing services unchanged on their original ports.
- WireGuard listening on `UDP 51820`.
- No process listening on or taking over ports `80`, `443`, `5777`, or `7777`.

Also test the applications locally:

```bash
curl -I http://127.0.0.1
curl -k -I https://127.0.0.1
```

For the proxy ports, use the appropriate protocol-specific check:

```bash
sudo nc -vz 127.0.0.1 5777
sudo nc -vz 127.0.0.1 7777
```

If anything stops working, take WireGuard down without rebooting:

```bash
sudo systemctl disable --now wg-quick@wg0
```

That removes the `PostUp` rules through the matching `PostDown` commands. Do not use broad commands such as `iptables -F`, `iptables -t nat -F`, or `ufw reset`; those could remove rules required by Apache, Xary, or WhatsApp Proxy.

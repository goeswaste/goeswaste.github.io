---
title: strongSwan deployment along side WireGuard
---

You can run strongSwan alongside WireGuard. For an iOS 6.1.4 device, use the built-in **Cisco IPSec** client, which means configuring strongSwan as an **IKEv1 + XAuth + pre-shared-key** remote-access server.

The older iOS client is not suitable for a modern IKEv2-only setup.

## 1. Install strongSwan

On a Debian/Ubuntu droplet:

```bash
sudo apt update
sudo apt install strongswan strongswan-pki libcharon-extra-plugins
```

The exact package names can vary by distribution. Ensure the XAuth plugin is installed:

```bash
sudo ipsec listplugins | grep -E 'xauth|mode-config'
```

You should have plugins comparable to:

```text
xauth-generic
attr
mode-config
kernel-netlink
```

If you use the newer `swanctl`/`charon-systemd` setup rather than legacy `ipsec.conf`, install the corresponding packages too. The example below uses the traditional `ipsec.conf` format because it is simpler for this legacy iOS client. strongSwan still supports IKEv1 for legacy compatibility, although IKEv1 is deprecated. <citation src="2,3"></citation>

## 2. Enable IP forwarding

```bash
sudo tee /etc/sysctl.d/99-vpn.conf >/dev/null <<'EOF'
net.ipv4.ip_forward=1
net.ipv4.conf.all.accept_redirects=0
net.ipv4.conf.all.send_redirects=0
EOF

sudo sysctl --system
```

Choose a private address pool that does not overlap with:

- The iPhone’s local Wi-Fi/mobile networks
- Your WireGuard address space
- Any networks you need to reach through the droplet

For example:

```text
iOS VPN pool: 10.20.0.0/24
WireGuard:    10.8.0.0/24
```

## 3. Configure `/etc/ipsec.conf`

Replace the following values:

- `YOUR_DROPLET_PUBLIC_IP`
- `YOUR_IOS_USERNAME`
- `YOUR_IOS_PASSWORD`
- `YOUR_GROUP_PASSWORD`

```conf
config setup
    uniqueids=no
    charondebug="ike 2, knl 2, cfg 2"

conn ios-cisco
    keyexchange=ikev1
    type=tunnel

    # Cisco IPSec on old iOS uses aggressive mode with XAuth.
    aggressive=yes
    authby=xauthpsk
    xauth=server

    # Phase 1: AES/SHA-1/DH group 2 is broadly compatible with old iOS.
    ike=aes128-sha1-modp1024

    # Phase 2.
    esp=aes128-sha1

    left=%defaultroute
    leftid=YOUR_DROPLET_PUBLIC_IP
    leftauth=psk

    # Give iOS a virtual address.
    leftsubnet=0.0.0.0/0
    right=%any
    rightauth=psk
    rightauth2=xauth
    rightsourceip=10.20.0.0/24

    # Full tunnel. Use a specific subnet here for split tunneling.
    rightsubnet=0.0.0.0/0

    dpdaction=clear
    dpddelay=30s
    rekey=no
    fragmentation=yes
    forceencaps=yes

    auto=add
```

For split tunneling, replace:

```conf
rightsubnet=0.0.0.0/0
```

with the networks the phone should access, for example:

```conf
rightsubnet=10.8.0.0/24
```

Do not use `leftsubnet=0.0.0.0/0` and `rightsubnet=0.0.0.0/0` if you only want access to selected private networks; broad selectors can interact poorly with existing routing and IPsec policies. strongSwan installs kernel IPsec policies separately from ordinary routes. <citation src="2"></citation>

## 4. Configure `/etc/ipsec.secrets`

```conf
: PSK "YOUR_GROUP_PASSWORD"

YOUR_IOS_USERNAME : XAUTH "YOUR_IOS_PASSWORD"
```

Example:

```conf
: PSK "replace-this-with-a-long-random-group-secret"

iosuser : XAUTH "replace-this-with-a-different-random-password"
```

The first secret is the Cisco IPSec group/shared secret. The second is the XAuth username and password. Cisco’s corresponding strongSwan example uses `authby=xauthpsk`, IKEv1, aggressive mode, XAuth, an assigned virtual address, and separate PSK/XAuth credentials. <citation src="3"></citation>

## 5. Enable the required strongSwan plugins

Check `/etc/strongswan.d/charon/xauth-generic.conf`. It should not be disabled:

```conf
xauth-generic {
    load = yes
}
```

Also check that the mode-config/attribute plugin is enabled. On many Debian-based systems the package defaults are already correct.

Restart strongSwan:

```bash
sudo systemctl restart strongswan-starter 2>/dev/null || \
sudo systemctl restart strongswan
```

Check its status:

```bash
sudo ipsec statusall
sudo journalctl -u strongswan-starter -f
```

## 6. Open the droplet firewall

Allow IKE, NAT-T, and ESP:

```bash
sudo ufw allow 500/udp
sudo ufw allow 4500/udp
sudo ufw allow proto esp
```

Also add equivalent rules in the cloud provider’s firewall/security-group panel.

The required traffic is:

```text
UDP 500   IKE
UDP 4500  IPsec NAT traversal
ESP       IP protocol 50
```

In practice, NAT-T normally carries the encrypted traffic through UDP 4500. Enabling `forceencaps=yes` helps when the phone or network is behind NAT.

Do not change WireGuard’s UDP port unless necessary. WireGuard and IPsec can coexist because WireGuard normally uses UDP 51820 while IKE uses UDP 500/4500.

## 7. Add forwarding and NAT rules

Assume:

- Public interface: `eth0`
- iOS pool: `10.20.0.0/24`
- WireGuard network: `10.8.0.0/24`

First identify your public interface:

```bash
ip route get 1.1.1.1
```

Then add forwarding rules:

```bash
sudo iptables -A FORWARD -s 10.20.0.0/24 -j ACCEPT
sudo iptables -A FORWARD -d 10.20.0.0/24 -j ACCEPT
sudo iptables -t nat -A POSTROUTING -s 10.20.0.0/24 -o eth0 -j MASQUERADE
```

If the iOS client must reach WireGuard peers:

```bash
sudo iptables -A FORWARD -s 10.20.0.0/24 -d 10.8.0.0/24 -j ACCEPT
sudo iptables -A FORWARD -s 10.8.0.0/24 -d 10.20.0.0/24 -j ACCEPT
```

You will also need a route or forwarding policy on the WireGuard side so that WireGuard peers know how to return traffic to `10.20.0.0/24`. With a typical WireGuard server configuration, add the iOS pool to the relevant peer’s `AllowedIPs`, or add an explicit route as appropriate.

Persist iptables rules using your distribution’s preferred mechanism, for example:

```bash
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

## 8. Configure iOS 6.1.4

On the iPhone:

1. Open **Settings → General → Network → VPN**.
2. Select **Add VPN Configuration**.
3. Choose **IPSec**.
4. Enter:

```text
Description:  My VPN
Server:       YOUR_DROPLET_PUBLIC_IP
Account:      YOUR_IOS_USERNAME
Password:     YOUR_IOS_PASSWORD
Use Certificate: Off
Group Name:   YOUR_GROUP_PASSWORD
Secret:       YOUR_GROUP_PASSWORD
```

On some iOS versions, the **Group Name** is an identifier while **Secret** is the shared secret. For the strongSwan configuration above, use the same group identifier/password arrangement consistently. If the device sends the group name as the IKE identity, set `leftid`/the PSK identity accordingly.

A commonly compatible arrangement is:

```text
Group Name:   vpn
Secret:       long-group-secret
Account:      iosuser
Password:     xauth-password
```

In that case, you may need to use the group identity in the secrets file:

```conf
vpn : PSK "long-group-secret"
iosuser : XAUTH "xauth-password"
```

## 9. Troubleshooting

Watch the logs while connecting:

```bash
sudo journalctl -f -u strongswan-starter
```

Or:

```bash
sudo ipsec statusall
```

Useful packet capture:

```bash
sudo tcpdump -ni any 'udp port 500 or udp port 4500 or proto 50'
```

Common failures:

- **`no proposal chosen`**: try `ike=aes128-sha1-modp1024` and `esp=aes128-sha1`.
- **IKE establishes but XAuth fails**: verify the XAuth username/password and `rightauth2=xauth`.
- **Authentication fails immediately**: verify the group secret and IKE identity/group name.
- **The tunnel connects but nothing is reachable**: check `ip_forward`, forwarding rules, NAT, and return routes.
- **Works on Wi-Fi but not cellular**: ensure UDP 4500 is open and keep `forceencaps=yes`.
- **No response at all**: check both the provider firewall and host firewall for UDP 500/4500.
- **Existing WireGuard routes interfere**: use non-overlapping pools and inspect:

```bash
ip route
sudo ip xfrm policy
sudo ip xfrm state
```

One important limitation is that iOS 6.1.4 and IKEv1/XAuth are obsolete and have weaker cryptographic capabilities than current clients. If you can upgrade the device or install a compatible third-party client, IKEv2 with certificates is preferable. The legacy Cisco IPSec method is useful here primarily because it is built into that old iOS release.

---
title: Deploy WireGuard on VPS
---

Assuming your Droplet runs **Ubuntu 22.04/24.04** and you want a full-tunnel VPN for one client, run this over SSH. It uses the standard WireGuard UDP port `51820`; allow that port in any DigitalOcean Cloud Firewall attached to the Droplet. <citation src="1"></citation>

### 1. Install WireGuard

```bash
sudo apt update
sudo apt install -y wireguard qrencode iptables
```

Enable IPv4 forwarding:

```bash
sudo tee /etc/sysctl.d/99-wireguard.conf >/dev/null <<'EOF'
net.ipv4.ip_forward=1
EOF

sudo sysctl --system
```

Find your public network interface:

```bash
ip route get 1.1.1.1
```

The interface is the device after `dev`, commonly `eth0` or `ens3`. Set it below:

```bash
export WAN_IF=eth0
```

### 2. Generate server and client keys

```bash
sudo mkdir -p /etc/wireguard
sudo chmod 700 /etc/wireguard
cd /etc/wireguard

sudo sh -c 'umask 077; wg genkey | tee server_private.key | wg pubkey > server_public.key'
sudo sh -c 'umask 077; wg genkey | tee client_private.key | wg pubkey > client_public.key'

sudo cat server_private.key
sudo cat server_public.key
sudo cat client_private.key
sudo cat client_public.key
```

Save the four displayed values temporarily. You will need them in the configuration.

### 3. Create the server configuration

Replace:

- `SERVER_PRIVATE_KEY`
- `SERVER_PUBLIC_KEY`
- `CLIENT_PUBLIC_KEY`
- `eth0` if your interface differs

```bash
sudo tee /etc/wireguard/wg0.conf >/dev/null <<EOF
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY

PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = CLIENT_PUBLIC_KEY
AllowedIPs = 10.8.0.2/32
EOF

sudo chmod 600 /etc/wireguard/wg0.conf
```

If your interface is `ens3`, replace both `eth0` occurrences:

```bash
sudo sed -i 's/eth0/ens3/g' /etc/wireguard/wg0.conf
```

### 4. Start WireGuard

```bash
sudo systemctl enable --now wg-quick@wg0
sudo wg show
```

You should see `interface: wg0` and UDP port `51820`.

### 5. Open the firewall port

If you use DigitalOcean Cloud Firewalls, add:

- **Type:** UDP
- **Port:** `51820`
- **Sources:** preferably your client’s public IP; use `0.0.0.0/0` if it changes frequently

If UFW is enabled on the Droplet:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 51820/udp
sudo ufw enable
```

### 6. Create the client configuration

On the server, retrieve the keys:

```bash
sudo cat /etc/wireguard/client_private.key
sudo cat /etc/wireguard/server_public.key
```

Create `client.conf` locally with:

```ini
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY
Address = 10.8.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = YOUR_DROPLET_PUBLIC_IP:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

Replace the three placeholders with the corresponding values. Import this file into the WireGuard app on your computer or phone.

To display a QR code directly in the terminal:

```bash
qrencode -t ansiutf8 < client.conf
```

Do not expose or share either private key.

### 7. Test the connection

After activating the client tunnel:

```bash
ping 10.8.0.1
```

On the Droplet, check the handshake:

```bash
sudo wg show
```

A recent `latest handshake` confirms that the client connected. From the client, visit an IP-checking website; it should show the Droplet’s public IP because `AllowedIPs = 0.0.0.0/0` routes all IPv4 traffic through the VPN.

If you only want access to the Droplet rather than a full-tunnel VPN, change the client setting to:

```ini
AllowedIPs = 10.8.0.0/24
```

and omit the NAT/forwarding setup.

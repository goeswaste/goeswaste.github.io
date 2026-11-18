---
title: how do i implement UDP2RAW to my Wireguard VPN?
date: 2026-11-18 00:08:22
---

The basic setup is a pair of local UDP relays: on the client, WireGuard sends packets to `127.0.0.1`; `udp2raw` carries them to the server, where it forwards them to WireGuard’s UDP port.

**Server:** keep WireGuard listening on UDP port `51820`, then run `udp2raw` to receive on port `443` and forward to WireGuard locally:

```bash
sudo udp2raw -s \
  -l 0.0.0.0:443 \
  -r 127.0.0.1:51820 \
  -k 'use-a-long-random-secret' \
  --raw-mode faketcp
```

Allow the chosen `udp2raw` port through the server’s firewall. The server’s WireGuard peer configuration should otherwise remain unchanged.

**Client:** run the client relay, forwarding to the server’s public address and port:

```bash
sudo udp2raw -c \
  -l 127.0.0.1:51821 \
  -r SERVER_PUBLIC_IP:443 \
  -k 'use-a-long-random-secret' \
  --raw-mode faketcp
```

In the client’s WireGuard peer section, change the endpoint to the local relay:

```ini
[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = 127.0.0.1:51821
AllowedIPs = ...
PersistentKeepalive = 25
```

Use the **same secret and raw mode at both ends**. Keep the actual WireGuard keys and `AllowedIPs` as usual. The `udp2raw` examples use this local-endpoint arrangement for WireGuard. <citation src="1,5"></citation>

If the tunnel connects but larger transfers stall, lower the WireGuard MTU on the client and test; `udp2raw` adds packet overhead, so the original MTU may be too large. One reported setup required lowering it, but the right value depends on the path. <citation src="3"></citation>

Run `udp2raw` as a managed service rather than relying on a shell command for long-term use. Use this only on networks and servers you’re authorized to administer, and check local network rules before deploying it.

---

Implementing UDP2RAW allows you to tunnel WireGuard's UDP traffic inside a fake TCP header. This is primarily used to bypass firewalls or ISPs that throttle or block UDP traffic (UDP blocking), as the traffic appears to the network as standard TCP.

Because UDP2RAW is a separate tool that acts as a wrapper, you must install and run it on both your WireGuard server and your WireGuard client.

### Prerequisites
- A working WireGuard installation on both server and client.
- Root or administrative access on both machines.
- The `udp2raw` binary installed on both ends.

### Step 1: Configure the Server
On your server, you need to tell UDP2RAW to listen for "fake TCP" packets and forward them to the actual WireGuard UDP port.

1. Run the UDP2RAW server command. Assuming your WireGuard port is `51820` and you want the fake TCP port to be `443`:
   `./udp2raw_amd64 -s -l 0.0.0.0:443 -r 127.0.0.1:51820 -k "your_password" --raw-mode faketcp`
   - `-s`: Server mode.
   - `-l`: The public-facing fake TCP port.
   - `-r`: The local WireGuard UDP port (using 127.0.0.1 ensures only the tunnel can access WireGuard).
   - `-k`: A shared password for authentication.

### Step 2: Configure the Client
The client must wrap the outgoing WireGuard UDP packets into fake TCP packets before they leave the device.

1. Run the UDP2RAW client command. Replace `SERVER_IP` with your server's actual IP:
   `./udp2raw_amd64 -c -l 127.0.0.1:51820 -r SERVER_IP:443 -k "your_password" --raw-mode faketcp`
   - `-c`: Client mode.
   - `-l`: A local UDP port that WireGuard will send traffic to.
   - `-r`: The server's fake TCP IP and port.

### Step 3: Update WireGuard Configuration
You must change your WireGuard client configuration to point to the local UDP2RAW tunnel instead of the remote server.

1. Edit your client `.conf` file.
2. Find the `Endpoint` line in the `[Peer]` section.
3. Change it from `Endpoint = SERVER_IP:51820` to `Endpoint = 127.0.0.1:51820`.

### Final Traffic Flow
The data path now looks like this:
**Client WireGuard App** $\rightarrow$ **Local UDP (127.0.0.1:51820)** $\rightarrow$ **UDP2RAW Client** $\rightarrow$ **Fake TCP (Internet)** $\rightarrow$ **UDP2RAW Server** $\rightarrow$ **Local UDP (127.0.0.1:51820)** $\rightarrow$ **Server WireGuard App**.

### Important Considerations
- **MTU Issues:** Because UDP2RAW adds a TCP header, the effective packet size decreases. You will likely need to lower the `MTU` in your WireGuard config (e.g., `MTU = 1200`) on both sides to prevent packet fragmentation and connection drops.
- **Persistence:** These commands run in the foreground. To make them permanent, you should create a `systemd` service unit (on Linux) to ensure UDP2RAW starts automatically on boot.
- **Firewalls:** Ensure that the fake TCP port (e.g., 443) is open in your server's firewall (ufw/iptables).

---

### how do i do this on an iOS device?

On a standard, non-jailbroken iPhone, you generally **can’t run `udp2raw` alongside the official WireGuard app**. The iOS app can’t point its tunnel at a local `127.0.0.1` relay in the way the desktop setup does, and `udp2raw` doesn’t provide a ready-to-use iOS client. The project’s discussion notes that iOS support would require implementing it as a VPN service, rather than simply running the existing relay. <citation src="1"></citation>

Your practical options are:

- **Run the client relay on another device**—such as a small router or computer—and connect the iPhone through that device’s Wi‑Fi.
- **Use a VPN protocol or service supported directly on iOS** that meets your needs, instead of `udp2raw`.
- **Build a custom iOS VPN app** that integrates the transport and WireGuard using Apple’s VPN APIs; that’s a software-development project, not an iPhone setting.

The official WireGuard app does support iOS, but that doesn’t add `udp2raw` transport support by itself. 

https://github.com/wangyu-/udp2raw-multiplatform/issues/56
---
title: Wireguard failed to complete handshake
date: 2026-11-16 07:43:23
---

The behavior you are describing—initial success followed by a complete dropout that resolves itself after several days—is a classic symptom of **GFW (Great Firewall) active probing and protocol fingerprinting**.

The GFW does not always block WireGuard instantly. Instead, it often uses a "detect and throttle/drop" approach. When it sees a sustained or periodic flow of UDP packets with the specific handshake patterns of WireGuard, it flags the VPS IP and begins dropping UDP packets to that destination. The "recovery" after a few days happens because the firewall's temporary blacklist for your IP expires.

Since WireGuard is a "silent" protocol (it doesn't respond to unauthorized packets), it is easy for the GFW to identify the distinct UDP patterns. To fix this, you must hide the WireGuard traffic.

### 1. Implement a UDP Obfuscation Layer
You cannot fix this by changing WireGuard settings alone because the protocol's header is static and recognizable. You need a "wrapper" that makes the traffic look like something else (or just random noise).

*   **Shadowsocks with UDP Relay:** Run a Shadowsocks server on your VPS and use a client that supports UDP relay. You then route your WireGuard traffic through the Shadowsocks tunnel.
*   **UDP2RAW:** This is a popular tool specifically for this scenario. It tunnels UDP packets through a fake TCP connection. Because the GFW is less likely to drop "TCP" traffic to a VPS than unexplained UDP traffic, this often bypasses the block.
*   **AmneziaWG:** This is a modified version of WireGuard specifically designed to bypass DPI (Deep Packet Inspection). It adds "junk" data to the handshake and changes the packet headers so the GFW cannot recognize the protocol. This is currently the most seamless "drop-in" replacement for standard WireGuard.

### 2. Use "Persistent Keepalive"
While this won't stop the GFW from detecting the protocol, it prevents the connection from timing out due to NAT mappings on your local network or iOS power management. 

In your iOS WireGuard config, set **Persistent keepalive** to `25`. This ensures the tunnel stays active during idle periods, though be aware that frequent keepalives can actually make it *easier* for the GFW to spot a constant heart-beat to a foreign VPS.

### 3. Change the Listen Port
Avoid using the default WireGuard port (`51820`). The GFW monitors common VPN ports more aggressively. Move your listen port to a common port that typically allows UDP, such as:
*   **53** (DNS)
*   **123** (NTP)
*   **443** (HTTPS/QUIC)

### Recommended Strategy
If you want the simplest path with the least amount of configuration overhead:

1.  **Switch to AmneziaWG.** It maintains the speed of WireGuard but disrupts the fingerprinting the GFW uses to identify and temporarily ban your IP.
2.  **Avoid "Idle" timeouts** by setting the keepalive to 25, but only *after* you have obfuscated the traffic.
3.  **Use a non-standard port** (e.g., 443 UDP) to blend in with HTTPS/QUIC traffic.
---
title: What can Tailscale do
date: 2026-11-16 00:59:54
---

Tailscale creates a private network—called a *tailnet*—that securely connects your devices over the internet. It’s built on WireGuard and usually lets devices connect directly, even when they’re behind different routers or firewalls, without setting up port forwarding. <citation src="2"></citation>

That lets you do things like:

- **Reach your own devices remotely:** connect to a home computer, NAS, Raspberry Pi, or media server while away.
- **Access private services:** securely reach a server, database, or development environment without exposing it publicly.
- **Use a device as an exit node:** route your internet traffic through a device on your tailnet, somewhat like a traditional VPN.
- **Connect a team’s devices and infrastructure:** give authorized people access to specific machines or services, with access rules.
- **Use Tailscale SSH:** connect to other tailnet machines without exposing SSH to the public internet. <citation src="2,3"></citation>

For personal use, common examples include a home lab, remote access to a home network, or playing games with friends. <citation src="3"></citation>

Tailscale is a Zero Trust networking platform that allows you to create a secure, private mesh network (called a **tailnet**) between your devices, regardless of where they are physically located. <citation src="3"></citation> It is built on top of the **WireGuard** protocol, providing end-to-end encryption and high-performance connectivity without requiring you to manage complex firewall rules or port forwarding. <citation src="1,3"></citation>

### Core Capabilities
*   **Secure Device Connectivity:** Connects laptops, servers, mobile devices, and IoT hardware into a single private network. <citation src="3"></citation>
*   **VPN Replacement:** Replaces traditional corporate VPNs by enabling direct, peer-to-peer encrypted connections between users and infrastructure, which reduces latency and removes central bottlenecks. <citation src="2,3"></citation>
*   **Exit Nodes:** Allows you to route all your internet traffic through a specific device on your network, similar to a commercial privacy VPN. <citation src="1"></citation>
*   **Subnet Routing:** Provides access to entire physical networks or legacy devices that cannot run the Tailscale client themselves. <citation src="1,5"></citation>
*   **Tailscale SSH:** Manages SSH connections using Tailscale's identity system, removing the need to manually distribute and manage SSH keys. <citation src="1"></citation>
*   **Taildrop:** Enables the secure transfer of large or sensitive files between your own devices. <citation src="1"></citation>
*   **Funnel:** Allows you to securely share a specific local service or folder with the public internet over HTTPS. <citation src="1"></citation>

### Common Use Cases
*   **Home Labs:** Accessing self-hosted services (like media servers or home automation) remotely without exposing them to the public internet. <citation src="3,5"></citation>
*   **Developer Workflows:** Sharing local development servers with teammates or testing webhooks without deploying to a cloud environment. <citation src="2,3"></citation>
*   **Enterprise Infrastructure:** Managing secure, audited access to production servers, Kubernetes clusters, and cloud VPCs. <citation src="2,3"></citation>
*   **AI & GPU Clusters:** Securing access to private LLM deployments and AI infrastructure. <citation src="2,3"></citation>
*   **Remote Management:** Using "Wake on LAN" or accessing remote machines across different continents. <citation src="5"></citation>
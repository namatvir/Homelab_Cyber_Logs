# Tailscale — Remote Access to the Pi

> Setup performed on personally owned Raspberry Pi 4 (Kali) and personal Mac, for personal remote access.

## Goal

Set up a way to reach the Pi (primarily via SSH) from any network, not just while on the same home wifi, without exposing any port directly to the internet via router port forwarding.

## Hardware / Software

- Raspberry Pi 4, Kali Linux
- Personal Mac
- Tailscale (built on WireGuard)

## Why this approach over the alternatives

A few options exist for remote access to a home device:

- **Port forwarding + dynamic DNS**: exposes a port (e.g. SSH) directly to the internet, which then gets scanned and brute-forced constantly. Requires hardening (key-only auth, non-default port, fail2ban) to be reasonably safe.
- **Reverse SSH tunnel through a VPS**: avoids exposing the home router directly, but adds a dependency on a separate always-on server and more moving parts.
- **Tailscale**: creates a private mesh network between authorized devices only. No port is ever opened to the public internet. Traffic is encrypted end to end and, where possible, routed directly between devices (peer-to-peer) rather than through a third party.

Given the goal was simple, low-maintenance remote access to a single personal device, Tailscale was the most direct fit.

## Setup

On the Pi:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo systemctl start tailscaled
sudo systemctl enable tailscaled
sudo tailscale up
```

`tailscale up` prints an authorization link. Opened it in a browser and logged in (GitHub account used as the identity provider) to authorize the Pi onto the private network.

On the Mac: installed the Tailscale app directly, logged in with the same account.

Confirmed both devices were connected and visible to each other:

```bash
tailscale status
```

## Verification

```bash
ssh kali@<pi-tailscale-hostname>
```

from the Mac, successful on the same network and confirmed again while on a separate network entirely (mobile data), confirming the connection is not dependent on both devices sharing a physical network.

## How it works, briefly

Both devices run a background service (`tailscaled`) that authenticates against Tailscale's coordination servers and negotiates a direct encrypted tunnel with other authorized devices. Each device is assigned a stable private address that does not change regardless of which physical network it is actually on. Tailscale is built on WireGuard, with the coordination layer handling key exchange and NAT traversal automatically, work that would otherwise have to be done manually between any two WireGuard peers.

Where a direct peer-to-peer connection isn't possible (for example, behind a restrictive firewall), traffic falls back to being relayed through Tailscale's own infrastructure, still fully encrypted, just not direct.

## Operational notes

- `tailscaled` must be enabled (`systemctl enable`), not just started, or it will not reconnect automatically after a reboot.
- The Pi still requires some internet connection for this to work at all. Tailscale solves "which network, what address," not "does the Pi have power or connectivity."
- The local network address (not the Tailscale address) should still be used for same-network traffic where latency matters or where the service doesn't need to be reachable from outside the home (for example, DNS queries to a local service), since routing local traffic through the tunnel unnecessarily is both slower and adds a dependency on Tailscale being up for something that doesn't need it.

## Takeaways

- Tailscale removes the two most annoying parts of traditional remote access setups: tracking a changing public IP, and exposing a port to the open internet. Neither the Pi's SSH port nor anything else had to be opened to the public internet at any point.
- The underlying mechanism (WireGuard plus a coordination layer for key exchange and NAT traversal) is worth understanding on its own, since it's the same protocol on the self-hosted VPN project planned separately for this repo, just with Tailscale handling the setup that would otherwise be manual.
- Remote reachability only solves the software and connectivity side of "access my Pi from anywhere." Hardware that physically needs to be near something (wifi adapters doing wifi-specific work) is still bound to wherever the Pi physically is; Tailscale does not change that.
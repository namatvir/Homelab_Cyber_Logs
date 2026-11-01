# Restricting SSH to a Tailscale Interface

## Summary

Two homelab machines run SSH only on their Tailscale addresses. SSH is unreachable from the local network and from the internet. This document covers the initial configuration and its verification. It also records a boot-time failure that this configuration introduced, which is fixed in `02-tailscale-ssh-boot-fix.md`.

## Environment

| Machine | OS | Role |
|---|---|---|
| Raspberry Pi 4 | Raspberry Pi OS Lite | Pi-hole (Docker) |
| Laptop | Debian | Uptime Kuma (Docker) |

Both machines are on a Tailscale tailnet. The router has no port forwards and does not allow custom DNS servers.

## Goal

Reduce SSH exposure to a single authenticated path. By default, `sshd` listens on every network interface. A future mistake such as an accidental port forward, a UPnP rule, or a router misconfiguration could expose SSH to the internet. Binding `sshd` to the Tailscale address removes that risk at the service level, independent of router configuration. This is defense in depth.

## Design Decision: Not Using Tailscale SSH

Tailscale offers a built-in SSH feature (`tailscale up --ssh`). I chose not to use it. A critical vulnerability was disclosed against it recently(known as the "Tailworm" vulnerability), and standard OpenSSH bound to a single interface has a smaller and better-understood attack surface for this use case.

## Implementation

On each machine, get the Tailscale IPv4 address:

```bash
tailscale ip -4
```

Add a `ListenAddress` directive using that machine's own address. I placed it in a drop-in file under `/etc/ssh/sshd_config.d/`, which the main `sshd_config` includes:

```
ListenAddress 100.x.x.x
```

Validate the configuration before applying it:

```bash
sudo sshd -t
```

Restart the service:

```bash
sudo systemctl restart ssh
```

## Verification

I kept the original SSH session open while testing from a second terminal, so a bad config could not lock me out.

1. Connecting to the Tailscale IP succeeded.
2. Connecting to the LAN IP was refused, because `sshd` is not listening on that address.

Listening sockets can also be checked directly:

```bash
sudo ss -tlnp | grep :22
```

The output should show port 22 bound only to the Tailscale address.

## Known Issue

After a reboot, SSH was unavailable (`connection refused`) until I ran `sudo systemctl start ssh` manually. The cause is a boot ordering race between `sshd` and Tailscale. The diagnosis and fix are in `02-tailscale-ssh-boot-fix.md`.

## Takeaways

Binding a service to an address that is created by another service introduces a startup dependency. Restricting exposure is worthwhile, but it needs to be tested across reboots, not only after a manual restart.
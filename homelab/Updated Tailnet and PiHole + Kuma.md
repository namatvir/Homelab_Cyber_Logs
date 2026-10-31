# Repurposed RPi and Ancient Gaming Laptop

> Setup performed on personally owned hardware (ASUS ROG laptop, Raspberry Pi 4, personal Mac), for a personal self-hosted services homelab.
> Reformatted Rpi4 with Pi OS lite as kali is unsuitable for 24/7 server

## Summary

First real build session of the homelab project. Repurposed a 10+ year old gaming laptop and an existing Raspberry Pi 4 (previously running Kali) into two dedicated, hardened, always-on servers connected over a private mesh network, with no ports exposed to the public internet. Deployed and verified two services end to end: network-wide DNS filtering and a status monitoring dashboard.

## Objective

Build the foundational layer of a self-hosted homelab: reliable always-on compute, secure remote access, and a small set of genuinely useful self-hosted services, before expanding into larger projects (media automation, a dedicated firewall/VLAN setup, a local AI assistant, etc.).

## Hardware

- ASUS ROG gaming laptop, 10+ years old, Intel Core i7, GTX 660M, two 500GB SATA SSDs. Repurposed as the primary always-on service host.
- Raspberry Pi 4, previously running Kali Linux for WiFi security work. Reflashed and repurposed as a dedicated DNS server.
- Personal Mac, used as the administration point for all SSH access and configuration.

## What I Did

**Ancient Laptop: OS install and hardening**
- Wiped Windows 10 and installed Debian 13 (Trixie), headless (no desktop environment).
- Created a non-root user with sudo privileges, separate password from the root account.
- Installed and configured Docker and Docker Compose.
- Restricted `sshd` to listen only on the Tailscale network interface, so SSH is unreachable from the local LAN or the public internet, only from authorized devices on the private mesh.

**Tailscale and Tailnet Lock**
- Installed Tailscale on all devices to create a single private mesh network (WireGuard-based), with no router port forwarding at any point.
- Enabled Tailnet Lock, designating the daily drivers as trusted signing nodes. This was done specifically in response to a disclosed (at time of writing, unpatched) Tailscale SSH vulnerability affecting networks without Tailnet Lock enabled.

**Services deployed on Ancient laptop via Docker**
- Pi-hole; network-wide DNS filtering. Initially deployed here as a temporary measure before planned migration to dedicated hardware.
- Uptime Kuma; status monitoring dashboard for all deployed services.

**Pi: reflash and hardening**
- Reflashed the Pi's SD card from Kali Linux to Raspberry Pi OS Lite (headless).
- Installed Docker and Docker Compose (required an alternate installation method after the architecture-specific package repository had no release available for the OS's current codename).
- Joined Pi to the Tailnet and restricted `sshd` to the Tailscale interface only, matching Io's configuration.
- Deployed Pi-hole on Pi, verified it resolved DNS correctly from the Mac before removing the Pihole containers from the gaming laptop.
- Decommissioned the Pi-hole instance on laptop via `docker compose down`, once the new instance was confirmed stable, and updated the Mac's DNS settings and the Uptime Kuma monitor to point at Pi instead.

## Problems Faced

- **SSH service not surviving reboot Pi.** After a clean install, SSH worked until the first reboot, then refused all connections. Root cause: the service had been started (`systemctl start`) but never enabled (`systemctl enable`) for automatic startup on boot. Required local console access (monitor and keyboard) to diagnose and fix, since the box was unreachable remotely by definition. Resolved by enabling the service properly; confirmed it persists across reboots going forward.
- **Accidental desktop environment install.** A software-selection screen during the Debian installer was confirmed with Enter instead of toggled with Space, resulting in a full GNOME desktop being installed on what was meant to be a headless server. Rather than fight dependency resolution to strip it out mid-install, did a clean reinstall with the correct package selection, resulting in a verified headless system.
- **Stale Docker Hub tag (Uptime Kuma).** The `:latest` tag for this specific image is deprecated and frozen on an old, unmaintained major version, contrary to what the tag name implies. Caught via an in-app warning after deployment, confirmed against the project's own documentation, and corrected by pinning to the maintainer-recommended `:2` tag instead. The general lesson is that a "latest" tag is not reliably current and needs to be checked against the project's own docs, not assumed from a Docker Hub search.
- **Docker install failure on laptop (architecture/codename mismatch).** The standard Docker APT repository had no published release for the Pi's OS codename (Debian Trixie) under the Raspbian repo path, causing repeated "no installation candidate" errors. Diagnosed via the `apt update` output showing a missing `Release` file, resolved by falling back to Docker's official install script (`get.docker.com`), which handled architecture and repository selection correctly where the manual method failed. A secondary complication during troubleshooting (a corrupted, duplicated line written into the APT source file from running a command twice) was identified and fixed by rewriting the file cleanly.
- **Pi SSH lockout after a WiFi credential change.** After changing WiFi credentials via `raspi-config` and losing the wired connection, the Pi became unreachable over both its local and Tailscale addresses. Diagnosis required local console access; cause was traced to a weak WiFi signal at the device's new physical location (upstairs, farther from the router) rather than a configuration error, as the same credentials and setup had worked correctly on a stronger connection. Given the device's role as always-on infrastructure, this reinforced a decision to prefer a wired connection for this box going forward.
- **Recurring SSH lockout, cause unconfirmed.** On more than one occasion (once after a WiFi credential change, once after moving the device to a new physical location), the Pi became unreachable over SSH on both its local and Tailscale addresses, requiring local console access (monitor and keyboard) to manually re-enable SSH before remote access worked again. The credentials and configuration were not the issue in at least one case, since they were unchanged from a prior working state. The exact root cause was not conclusively identified during this session; a weak WiFi signal at the device's relocated position was considered but not confirmed as the cause. Given the device's role as always-on infrastructure, this is flagged as an open reliability issue worth root-causing properly, and reinforced a decision to prefer a wired connection for this box going forward regardless.
- **Accidental exposure risk from Tailscale's built-in SSH feature.** While troubleshooting the above lockout, `tailscale set --ssh` was run, enabling Tailscale's own SSH server, a separate system from the already-configured and hardened OpenSSH service, with its own, unconfigured access-control policy. This caused a second, unrelated lockout (an access-denied error from Tailscale's policy engine) and was a deliberate thing to avoid, given prior research into a disclosed Tailscale SSH vulnerability. Resolved by disabling the feature (`tailscale set --ssh=false`) and confirming the existing, properly hardened OpenSSH path still worked as intended.
- **Raspberry Pi Imager's SSH-enable file alone was insufficient.** Manually placing an empty `ssh` file on the boot partition enables the SSH service but does not create a user account on current Raspberry Pi OS images, which no longer ship a default login. Diagnosed after repeated "connection refused" errors with no valid credentials to try, resolved by using Raspberry Pi Imager's OS customization options to set a real username, password, and WiFi configuration before flashing, rather than relying on the legacy manual-file method.

## Overall Layout

- Home router: no ports forwarded, no public attack surface.
- Tailscale mesh (WireGuard-based): the only remote access path between all devices, reachable from any network without exposing anything to the internet. Tailnet Lock enabled, Mac and phone as trusted signers.
- **Laptop** (Debian, headless): Docker host for future projects (referenced in future write-ups), Uptime Kuma. SSH restricted to the Tailscale interface, non-root sudo user.
- **Pi** (Raspberry Pi OS Lite, headless): dedicated DNS server running Pi-hole in Docker. Same SSH hardening as Laptop.
- Both machines monitored from a single Uptime Kuma dashboard on Laptop.

## Future Goals (brief)

- Full Jellyfin build-out: VPN-gated torrent automation stack, pointed strictly at legally owned or public-domain sources, plus a dedicated music client.
- A private tracker for personal finances, investments, bills and income.
- A second private tracker for ongoing projects and coursework.
- A local AI chat interface, with the actual model inference running on separate, more capable hardware.
- Eventual purchases to support the next phase: a dedicated NAS for photo/video storage, a small low-power machine to take over always-on duties from the current laptop, and a managed switch to support real network segmentation.

## Takeaways

- A huge proportion of today's actual time went into debugging, not initial setup, and nearly every failure was a known, well-documented class of problem (`enable` vs `start`, a stale Docker tag, an architecture/repo mismatch, a legacy setup method no longer matching current defaults) rather than anything exotic. Recognizing the category of problem mattered more than any single fix.
- Verifying a new service (new OS install, new Docker image, new config change) before trusting it, rather than assuming it worked, caught real issues early, the Uptime Kuma tag warning and the Pi-hole migration's "test before cutover" approach both paid off directly.
- Hardening decisions made early (SSH restricted to Tailscale only, Tailnet Lock, non-root operation) held up under real conditions today, including an accidental misconfiguration that could have otherwise left a second, unintended SSH path open.
- Infrastructure that needs to be reliably always-on has different requirements than infrastructure used occasionally; an intermittent SSH access issue with an unconfirmed cause is tolerable on a personal laptop but a real liability on a device acting as the network's only DNS resolver, worth root-causing and designing around rather than accepting as a one-off.
# Pi-hole — Network-Wide DNS Filtering via Docker
 
> Setup performed on personally owned Raspberry Pi 4 (Kali), for personal home network use.
 
## Goal
 
Stand up Pi-hole as a Docker container on the Pi to provide DNS-level ad/tracker blocking, first tested against a single device (Mac) via manual DNS configuration, since the home ISP's gateway blocks router-level DNS overrides.
 
## Hardware / Software
 
- Raspberry Pi 4, Kali Linux
- Docker (installed manually after resolving a broken `kali-rolling` repo reference; Docker's install script assumes a Debian codename that doesn't exist for Kali, fixed by manually pointing `/etc/apt/sources.list.d/docker.list` at `bookworm`)
- `pihole/pihole:latest` Docker image
## Setup
 
```bash
sudo docker run -d \
  --name pihole \
  -p 53:53/tcp -p 53:53/udp \
  -p 80:80 \
  -e TZ="America/Toronto" \
  -v "$(pwd)/pihole/etc-pihole:/etc/pihole" \
  -v "$(pwd)/pihole/etc-dnsmasq.d:/etc/dnsmasq.d" \
  --restart=unless-stopped \
  pihole/pihole:latest
```
 
Retrieved/set the admin password:
```bash
sudo docker exec -it pihole pihole setpassword
```
 
Confirmed the container and dashboard worked (`docker ps`, logged into the web UI on the Pi's local address) before attempting to actually route any real traffic through it.
 
## Router-level DNS: blocked
 
Attempted to set Pi-hole as the network-wide DNS server via router DHCP settings (the intended, set-it-once, works-for-every-device approach). The ISP's gateway does not expose a DNS override option at all. Confirmed this is an ISP-enforced restriction, not a missing menu. Fallback: per-device DNS configuration instead of network-wide, with proper bridge mode and a personal router as a separate, larger project for true network-wide filtering later.
 
## Per-device setup (Mac) and the real issue
 
Set the Pi's local address as primary DNS and a public fallback resolver as secondary in macOS Network settings. Initial test appeared to work (ads briefly missing) but was actually a false positive, likely stale cached DNS entries or browser-level blocking, not Pi-hole. Removing the fallback entirely broke DNS resolution completely, confirming queries were never actually reaching Pi-hole in the first place.
 
### Debugging sequence
 
1. Queried Pi-hole via loopback on the Pi itself. Succeeded, clean answer. Confirmed Pi-hole itself was healthy.
2. Queried Pi-hole via its real local address from both the Pi and the Mac. Timed out both times. Confirmed the problem was specific to reaching Pi-hole via its real network-facing address, not Pi-hole's core function.
3. Checked for a host-level firewall blocking port 53. `iptables` showed an ACCEPT policy on the INPUT chain, ruling out a simple default-deny block.
4. Found all INPUT traffic routed through a chain inserted by Tailscale's own iptables integration. Inspected it directly; no rule affecting port 53 or general LAN traffic. Ruled out as the cause.
5. Confirmed general Mac-to-Pi connectivity was fully healthy (a basic TCP connectivity test to the dashboard's port succeeded), narrowing the issue specifically to port 53.
6. Ran a packet capture on the Pi's wifi interface, filtered to port 53, while querying from the Mac. Queries were arriving at the Pi (visible in the capture, including background traffic from the Mac's own OS and apps), but no response was ever sent back. This ruled out any network-path or firewall block entirely; the issue was inside Pi-hole's own request handling.
7. Checked Pi-hole's container logs and found the actual cause: a warning indicating `dnsmasq` was ignoring queries from a "non-local network." Pi-hole's underlying `dnsmasq` engine was configured with a restrictive listening mode, which by default under Docker only trusts Docker's internal subnet, not the actual home LAN subnet the Mac was querying from. The Mac's queries were being silently dropped at the application layer, despite arriving successfully at the OS/network level.
### Fix
 
```bash
sudo docker exec pihole pihole-FTL --config dns.listeningMode ALL
sudo docker restart pihole
```
 
Re-tested from the Mac; immediate, clean response.
 
## Verification
 
- Query to a normal domain from the Mac resolves correctly
- Query to a known ad-serving domain from the Mac returns a blocked response, confirming the filtering logic itself discriminates correctly rather than just passing or failing wholesale
- Pi-hole dashboard query log populated with real-time traffic from the Mac, including blocked entries
- Dashboard summary stats (queries today, percent blocked) incrementing live while browsing
## Takeaways
 
- A packet arriving successfully at a host (confirmed via packet capture) does not mean the application listening on that port will accept or respond to it. Host-level network debugging (firewall rules, routing, basic connectivity tests) can all check out clean while the actual service still silently drops the request at its own application-layer filtering logic. Worth checking the service's own logs early, not just the network path, when something should be reachable but isn't.
- Docker's default `dnsmasq` listening behavior under Pi-hole is scoped to trust only the Docker-internal network by default, not the host's real LAN. This is a known gap between "container is running and port is published" and "service will actually answer queries from the host's real network," worth setting explicitly on any Dockerized DNS service rather than assuming default Docker port publishing is sufficient.
- A UDP connectivity test without a real handshake is not reliable confirmation that anything is listening or responding on the other end. A live packet capture is the only way to get a real answer when debugging UDP-based services like DNS.
- ISP-level restrictions on router DNS settings are a real, hard limitation for network-wide ad blocking on some providers. A personal router in bridge mode in front of the ISP gateway is the correct long-term fix, not something to improvise around at the per-device level indefinitely.
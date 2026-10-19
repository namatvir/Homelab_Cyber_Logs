# Evil Twin AP — Raspberry Pi 4, Dual Adapter

> Network details in this writeup are genericized. This was performed against a personally owned router/device, with explicit authorization; no third-party network or device was targeted.

## Goal

Stand up a rogue AP cloning a personally owned network's SSID, confirm a personal test device associates with it instead of the legitimate AP, and observe client traffic hitting a Pi-controlled interface, to validate the AP-impersonation side of the 802.11 trust model (as opposed to the client-capture work done in the handshake writeup above).

## Hardware / Software

- Raspberry Pi 4, Kali Linux
- Two wifi adapters: onboard/primary (`wlan0`, reserved for SSH/management) and a secondary adapter (`wlan1`, used for the attack. specficially an Alfa AWUS036ACM Wifi Adapter)
- `airbase-ng` (aircrack-ng suite), `dnsmasq`, `tcpdump`
- Target: personally owned WPA2-PSK router's SSID (cloned, not attacked directly), 2.4GHz

## Setup

Interface separation confirmed first, since this attack reconfigures the attack adapter repeatedly and a single-adapter setup would risk severing the SSH session used to control the Pi:

```bash
ip a   # confirm wlan0 and wlan1 are separate physical adapters
       # note wlan0's subnet to avoid collision with the fake AP's subnet later
```

## Launching the rogue AP

```bash
sudo airmon-ng stop wlan1mon      # attack interface must NOT be in monitor mode for airbase-ng
sudo airbase-ng -e "<SSID>" -c <channel> wlan1
```

This creates a new virtual interface, `at0`, which carries the fake AP's data path. Left running in foreground for the duration of the test.

## Network configuration

```bash
sudo ifconfig at0 up
sudo ifconfig at0 192.168.1.1 netmask 255.255.255.0
```

Subnet chosen deliberately distinct from `wlan0`'s to avoid two interfaces on the Pi claiming overlapping local address space.

```bash
# dnsmasq.conf
interface=at0
dhcp-range=192.168.1.10,192.168.1.100,255.255.255.0,12h
```

```bash
sudo dnsmasq -C dnsmasq.conf -d
```

## Observation

```bash
tcpdump -i at0
```

## Result

Test device associated with the rogue AP without a deauth being necessary — the AP was broadcast open (no password), and association appears to have been driven by a combination of signal strength and the client device picking an available network matching a known SSID. Devices generally do not verify AP identity beyond SSID and security type, so there was nothing on the client side to distinguish the clone from the real AP.

`dnsmasq` logged a full DHCPDISCOVER → DHCPOFFER → DHCPACK sequence, confirming the client received a valid lease in the configured pool. `tcpdump` on `at0` showed outbound traffic from the client (DNS lookups, ARP requests, background app connectivity checks) consistent with a device that believes it has a working network connection.

`at0` had no onward route to the internet by design — all observed traffic terminated at the Pi rather than reaching its actual destination. This was a deliberate scope limit for this test, not a failure: the goal was confirming association and passive visibility, not full interception.

## What wasn't done

- No IP forwarding / NAT out through `wlan0` — client had no real internet access, so no payload data was ever actually transmitted to capture
- No captive portal or credential harvesting page
- No deauth used against the legitimate AP to force the client over; association happened without it

These represent the next logical steps if this is extended, not oversights in this test.

## Takeaways

- Client devices pick networks primarily on SSID + security type, not verified AP identity — this is a protocol-level gap, not something specific to any one router or device.
- An open (no-password) rogue AP will often win association over a legitimate password-protected one with no additional effort, particularly at close range — no deauth required in this test.
- A rogue AP with no forwarding configured is a legitimate and useful intermediate test state: it proves association and gives visibility into attempted connections without yet building out full interception.
- `tcpdump -i at0` is the right place to watch for live confirmation of a client landing on the fake AP — faster feedback than relying on the client device's own UI, which may not clearly indicate anything is wrong.
- Two separate wifi adapters are effectively required for this attack if a management/SSH session needs to stay alive on the same Pi — the attack interface gets reconfigured (monitor mode off, AP mode, new virtual interface) in ways that would otherwise kill the control connection.

## Disclaimer

Performed entirely against own hardware (personal AP clone, personal test device) in a home lab for educational purposes. Not tested against, or intended for use against, any network or device without explicit authorization.

## Reproduced Results

Repeat run against the same target: the test device initially associated with the legitimate AP over the clone despite the rogue AP being up and broadcasting, and this held across several wifi off/on retries. Note the Pi and real router were colocated, so eventual success on this run wasn't due to improved relative signal strength; cause is unconfirmed. On the connection that did succeed, the device also showed an "insecure network" warning (security type mismatch against the known SSID), which had not appeared on an earlier successful connection in this same run, suggesting the check isn't applied consistently. The warning appeared after association had already occurred, not before.
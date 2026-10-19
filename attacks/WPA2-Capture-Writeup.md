# WPA2 Handshake Capture — Raspberry Pi 4 + Alfa (MT7612U)

> Network details in this writeup are genericized. This was performed against a personally owned router/device, with explicit authorization; no third-party network or device was targeted.

## Goal

Capture a full WPA2 4-way handshake from a home test network using a Raspberry Pi 4 and an Alfa MT7612U (802.11ac) adapter, to validate the full monitor-mode → injection → capture pipeline end to end.

## Hardware / Software

- Raspberry Pi 4, Kali Linux
- Alfa adapter, MediaTek MT7612U chipset (`mt76x2u` driver, in-kernel)
- aircrack-ng suite (`airmon-ng`, `airodump-ng`, `aireplay-ng`, `aircrack-ng`)
- Target: personally owned WPA2-PSK router, 2.4GHz, channel 11

## Setup

```bash
sudo airmon-ng check kill
sudo airmon-ng start wlan1
sudo airmon-ng   # confirm wlan1mon is bound to mt76x2u and in monitor mode
```

## Reconnaissance

```bash
sudo airodump-ng wlan1mon
```

Identified target BSSID, channel, and encryption type from the AP table.

## Targeted capture

```bash
sudo airodump-ng -c 11 --bssid <BSSID> -w capture wlan1mon
```

## Deauth attempt

```bash
sudo aireplay-ng -0 5 -a <BSSID> -c <client MAC> --ignore-negative-one wlan1mon
```

Deauth packets were sent and acknowledged, but the target client did not visibly drop association — consistent with Protected Management Frames (802.11w/PMF) being enabled on the AP, which cryptographically protects against spoofed deauth frames. This wasn't confirmed directly (no admin access to the router at the time) but is the most likely explanation given the observed behavior.

## What actually worked

Deauth being ineffective, the handshake was instead captured by forcing a **legitimate** reassociation: forgetting the network on the client device and manually rejoining with the password while the targeted capture was running.

Initial attempts still failed. Inspecting the capture in Wireshark/tshark (`eapol` filter) showed only **messages 1 and 3** of the handshake (both AP → client) — messages 2 and 4 (client → AP) were never captured. This pointed to a one-directional RF issue: the AP's stronger transmit signal was being received fine, but the client's weaker transmit power wasn't reaching the adapter reliably at the distance/positioning used.

Moving the client device physically next to the Alfa antenna and repeating the forget/rejoin resolved it — all 4 EAPOL messages were captured.

## Verification

```bash
aircrack-ng capture-01.cap
```

Confirmed `1 handshake` present for the target BSSID.

## Takeaways

- A visible AP with strong signal doesn't guarantee a client's frames are being captured — check both directions independently when a handshake won't land.
- `tshark -r <file>.cap -Y eapol` is a more reliable way to diagnose a partial handshake than trusting `airodump-ng`'s live on-screen handshake indicator, which can be inconsistent.
- Deauth attacks are not guaranteed to work against modern APs/clients with PMF enabled; a legitimate forced reassociation is a reliable fallback when authorized to do so directly on the client device.
- Client transmit power is often the weaker link in range — position test devices accordingly.
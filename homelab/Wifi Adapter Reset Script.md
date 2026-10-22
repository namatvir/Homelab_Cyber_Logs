# Diagnosing Alfa AWUS036ACM (mt76x2u) Instability in Monitor Mode

> Performed on personally owned hardware, Raspberry Pi 4 and Alfa AWUS036ACM wifi adapter.

## Goal

Diagnose and resolve intermittent failures during wifi auditing sessions (handshake capture hanging or failing inconsistently, sometimes mid-session) on an Alfa AWUS036ACM adapter (MediaTek MT7612U chipset, `mt76x2u` driver), after ruling out power delivery as the cause.

## Hardware / Software

- Raspberry Pi 4, Kali Linux
- Alfa AWUS036ACM, MediaTek MT7612U chipset, `mt76x2u` driver (in-kernel)
- `aircrack-ng` suite, `wifite`

## Symptoms

- Manual handshake capture and `wifite` runs that worked cleanly in earlier sessions became unreliable over time: capture would stall indefinitely with no clear error, or wifite's automated flow would hang.
- A full physical unplug and replug of the adapter consistently and immediately resolved the issue, every time it was tried.
- A software-level Pi reboot (`sudo reboot`) alone did **not** reliably fix it; a full physical power cycle of the adapter did.

## Ruling out power delivery

Initial hypothesis was insufficient power to the Pi itself. Checked directly:

```bash
vcgencmd get_throttled
```

Result: `throttled=0x0`, consistently, across multiple sessions including ones where the adapter had just failed. This is a clean result indicating no undervoltage or thermal throttling has occurred since boot. Power delivery to the Pi as a whole was ruled out as the cause.

Also tested moving the adapter from a USB 3.0 port to a USB 2.0 port, which some community reports for this chipset associate with better monitor-mode stability. This did not eliminate the underlying issue on its own, though it did not appear to make things worse either.

## Actual cause

Research (cross-checked against real community bug reports) confirmed this is a known, recurring issue with MediaTek `mt76`-family USB wifi chipsets under sustained monitor mode or active packet injection. The pattern, confirmed across multiple independent sources (OpenWrt's `mt76` issue tracker, the morrownr USB-WiFi community, and others), is:

- Under sustained monitor-mode use, especially with injection, the adapter's firmware can enter a stuck state, commonly surfacing in kernel logs as an MCU (microcontroller unit) timeout.
- Once this happens, the driver generally cannot recover the firmware in software; only a full USB re-enumeration restores the adapter.
- A full physical unplug and replug forces this re-enumeration, which is why it reliably fixed the issue, while a plain OS reboot, which does not necessarily reset USB bus/device state, did not.

Some specific technical details found in initial research (an exact kernel regression commit hash and exact affected kernel version range) could not be independently verified and are not repeated here as fact. The general phenomenon (MCU timeout, firmware stall, fix via re-enumeration) is well corroborated; the specific unverified details were discarded rather than treated as confirmed.

## Fix: software-triggered USB re-enumeration

Rather than requiring a physical replug each time, the same re-enumeration can be triggered through sysfs directly, recovering the adapter without physically touching it.

Identify the adapter's USB path:
```bash
lsusb -t
```

Confirm the matching sysfs device entry corresponds to the adapter (vendor ID `0e8d`, product ID `7612` for the MT7612U):
```bash
cat /sys/bus/usb/devices/<device-path>/idVendor
cat /sys/bus/usb/devices/<device-path>/idProduct
```

Trigger the reset:
```bash
echo '<device-path>' | sudo tee /sys/bus/usb/drivers/usb/unbind
sleep 2
echo '<device-path>' | sudo tee /sys/bus/usb/drivers/usb/bind
```

Confirm recovery:
```bash
lsusb
iwconfig
```

This was wrapped into a short reusable script for faster recovery going forward:

```bash
#!/bin/bash
echo '<device-path>' | sudo tee /sys/bus/usb/drivers/usb/unbind
sleep 2
echo '<device-path>' | sudo tee /sys/bus/usb/drivers/usb/bind
echo "Adapter reset complete"
```

## Additional mitigation tried

Disabling USB scatter-gather for the `mt76_usb` module, referenced in community reports as improving stability for this chipset family, though reported success varies by kernel version:

```bash
echo "options mt76_usb disable_usb_sg=1" | sudo tee /etc/modprobe.d/mt76_usb.conf
sudo reboot
```

## Takeaways

- A symptom that looks like a power problem (device becomes unresponsive, recovers after a physical unplug/replug) is not always a power delivery problem. `vcgencmd get_throttled` gives a definitive, hardware-reported answer on Pi power state and should be checked directly rather than assumed from secondhand symptoms.
- A software reboot and a physical power cycle are not equivalent: a reboot restarts the OS, but does not necessarily reset USB bus or device firmware state. Some failure modes only clear with genuine USB re-enumeration.
- This specific chipset family (`mt76x2u`/MediaTek MT7612U) has a real, independently documented history of MCU timeout and firmware stability issues under sustained monitor mode or injection load. This is a known hardware/driver limitation, not a misconfiguration specific to this setup.
- A software-triggered USB re-enumeration via sysfs unbind/bind achieves the same recovery as a physical replug, without requiring physical access to the device, useful for recovering a remote or hard-to-reach setup.
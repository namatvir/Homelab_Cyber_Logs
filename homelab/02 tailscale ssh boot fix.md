# Fixing SSH Failing at Boot When Bound to a Tailscale IP

## Summary

After restricting `sshd` to a Tailscale address (see `01-tailscale-ssh-hardening.md`), SSH was unavailable after every reboot. The cause was a boot-time race between `sshd` and Tailscale. Setting the kernel parameter `net.ipv4.ip_nonlocal_bind=1` fixed it on both machines without changing the SSH security model.

## Symptom

After a reboot or power cycle, SSH to the Tailscale IP returned `connection refused`. Running the following on the machine locally restored access:

```bash
sudo systemctl start ssh
```

The `systemctl enable ssh` I had also been running was irrelevant. The service was already enabled. The `start` was what worked.

## Diagnosis

The SSH unit's logs for the current boot showed the failure:

```bash
journalctl -u ssh -b | grep -i "cannot assign"
```

```
sshd[828]: error: Bind to port 22 on <tailscale-ip> failed: Cannot assign requested address.
```

The sequence at boot was:

1. systemd starts services in parallel, and `sshd` started before Tailscale.
2. The Tailscale IP did not exist yet, because the `tailscale0` interface had not been created.
3. `sshd` tried to bind to an address the machine did not have, got `Cannot assign requested address`, and exited.
4. Tailscale came up a few seconds later, but `sshd` was already dead.
5. Starting `sshd` manually worked because the address existed by then.

I also checked `systemctl status ssh.socket`. It was disabled and inactive, so socket activation was not involved and `ListenAddress` was being honored normally. The position of the directive in the config file was not a factor.

## Fix

Allow the kernel to bind to addresses that are not yet assigned. On each machine:

```bash
echo 'net.ipv4.ip_nonlocal_bind=1' | sudo tee /etc/sysctl.d/99-nonlocal.conf
sudo sysctl --system
```

The first command writes a persistent setting that loads at every boot. Any `.conf` file in `/etc/sysctl.d/` is read, and the `99` prefix makes it load last. The second command applies it immediately without a reboot.

With the setting enabled, `sshd` binds to the Tailscale IP at boot even though the interface does not exist yet. Connections start succeeding as soon as Tailscale creates the interface. The race still occurs, but it no longer has any effect.

## Verification

I rebooted each machine and connected over the Tailscale IP without any manual intervention. SSH was available on its own after boot. I kept physical access available during the test in case the fix did not work.

## Security Impact

None for the SSH restriction. `sshd` still listens only on the Tailscale address and still refuses LAN connections. The setting only removes the kernel check on whether a bind address exists.

One caveat: the setting is system-wide. It applies to every program on the machine, not only `sshd`. A service with a mistyped bind address will now start without error and simply not receive traffic. For a homelab this is an acceptable tradeoff, but it is worth knowing when debugging.

## Alternatives Considered

**Systemd ordering (`After=tailscaled.service`).** Adding this to a drop-in override for the SSH unit is the conventional approach. It is less reliable here, because `tailscaled` being started does not guarantee the interface has its IP assigned yet. The race would be narrowed but not removed.

**Firewall rule instead of `ListenAddress`.** Let `sshd` listen on all interfaces and allow port 22 only on `tailscale0`. This avoids the race entirely and is arguably the more robust design. I deferred it because it adds firewall management to a setup that does not need it yet. On the Pi it would also require explicit allowances for DNS and the Pi-hole admin ports, or Pi-hole would break.

**Tailscale SSH.** Rejected for the security reason documented in the first write-up.

## Takeaways

When a service depends on an address created by another service, test the setup across a cold reboot. A configuration that works after a manual restart can still fail at boot. Reading the unit's boot log (`journalctl -u <unit> -b`) found the root cause in one command and ruled out guesswork.
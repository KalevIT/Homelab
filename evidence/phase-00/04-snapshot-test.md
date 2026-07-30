# Phase 0 — Exit criterion 4

**Requirement:** snapshot taken, virtual machine deliberately broken, snapshot
restored successfully.

**Date:** 2026-07-29

---

## Why this test exists

The criterion does not ask for a snapshot. It asks for a snapshot that has been
*restored*. Until a restore has been observed working, the lab has the belief in a
safety net rather than the safety net itself — and the difference only becomes
apparent at the worst possible moment.

## Snapshot created

| Field | Value |
|---|---|
| Name | `clean-install-20260729` |
| Created | 2026-07-29 21:36 |

Description recorded in the snapshot:

```
Ubuntu Server 26.04 LTS, standard install.
apt update + full-upgrade applied. Reboot completed.
OpenSSH server installed, password authentication still enabled.
No hardening applied yet. No SSH keys configured.
```

The description states what the machine *returns to*, not what was done to it. The
last two lines are the most useful: they record what is absent.

## Fault introduced

`/etc/fstab` was chosen because it controls which filesystems are mounted at boot.
An invalid entry prevents startup, and a malformed line is one of the most common
causes of boot failure on real systems.

```bash
echo "/dev/sdz1 /mnt/rotto ext4 defaults 0 2" | sudo tee -a /etc/fstab
tail -3 /etc/fstab
```

Confirmed appended:

```
/dev/sdz1 /mnt/rotto ext4 defaults 0 2
```

Then `sudo reboot`.

## Observed failure

Boot stalled waiting for a device that does not exist, then dropped to emergency
mode:

```
[  OK  ] Started emergency.service - Emergency Shell.
[  OK  ] Reached target emergency.target - Emergency Mode.
[  OK  ] Stopped target ssh-access.target - SSH Access Available.

You are in emergency mode. After logging in, type "journalctl -xb" to view
system logs, "systemctl reboot" to reboot, or "exit" to continue bootup.
Press Enter for system maintenance
(or press Control-D to continue):
```

Worth noting from the boot output: `ssh-access.target` was explicitly *stopped*. A
machine in this state is unreachable over the network, which is exactly why the
recovery path has to be tested before it is needed.

No repair was attempted. Diagnosis was not the objective of this test.

## Restore

VMware Snapshot Manager → `clean-install-20260729` → **Go To**.

The machine returned to the pre-fault state. Verification after login:

```bash
tail -3 /etc/fstab
```

```
# /boot was on /dev/sda2 during curtin installation
/dev/disk/by-uuid/1f8df51d-... /boot ext4 defaults 0 1
/swap.img       none    swap    sw      0       0
```

The `/dev/sdz1` entry is gone. System information reports the same disk usage
(42.7%) and the same address (192.168.231.129) as before the fault.

**Criterion met.** The restore path is proven, not assumed.

---

## Incidental lesson

The real entries in this `fstab` do not use device names. They use
`/dev/disk/by-uuid/...` and `/dev/disk/by-id/dm-uuid-LVM-...`, and the file's own
header comment explains why:

> `Use 'blkid' to print the universally unique identifier for a device; this may be
> used with UUID= as a more robust way to name devices that works even if disks are
> added and removed.`

The fault line used `/dev/sdz1` — a device path. Device names are assigned in
detection order and can change when hardware is added, removed or simply enumerated
differently. Identifiers belong to the filesystem and do not move.

Ubuntu's installer already applies this. Encountering the failure mode first, and
then noticing that the default configuration was built to avoid it, is a more durable
way to learn the rule than reading it.

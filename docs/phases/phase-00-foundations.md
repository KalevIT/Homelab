# Phase 0 — Linux and CLI Foundations

**Concurrent VMs:** 1 | **RAM budget:** 4 GB | **Disk:** ~25 GB
**Estimated effort:** 12-15 hours over ~3 weeks at 5 h/week
**Status:** In progress

---

## Objective

Reach the point where a terminal is a working environment, not an obstacle.

Nothing in this phase is security tooling. No firewall, no SIEM, no attack tools. Every
later phase assumes the shell is transparent, and building on top of an untested
assumption is how home labs collapse in month three.

**Prior knowledge:** LPI Linux Essentials (theory). The concepts are known; the muscle
memory is not. This phase converts one into the other, which is why the estimate is
compressed rather than extended.

---

## Step 1 — Correct the existing virtual network configuration

Do this before creating any virtual machine.

The networks were reset to VMware defaults, so VMnet2 must be created rather than
corrected.

**Edit → Virtual Network Editor → Change Settings** (requires administrator rights)

| Network | Action | Host virtual adapter | Local DHCP service |
|---|---|---|---|
| VMnet0 (bridged) | Leave alone | n/a | n/a |
| VMnet1 (host-only) | Leave as default | Checked | Checked |
| VMnet8 (NAT) | Leave as default | Checked | Checked |
| **VMnet2** | **Add Network** → Host-only | **Uncheck** | **Uncheck** |

For VMnet2, set Subnet IP `10.10.10.0` and mask `255.255.255.0`. Selecting the
host-only type re-checks the host adapter box automatically — clear it afterwards and
confirm it stays cleared after **Apply**.

Reasoning is documented in [`../lab-architecture.md`](../lab-architecture.md).

**Verification:** run `ipconfig` on the Windows host. Adapters for VMnet1 and VMnet8
must be present. **No adapter for VMnet2 may exist.** Save the output to
`evidence/phase-00/01-host-adapters.txt` — redacted per the sanitisation policy.

### Disconnect the VPN client

The host VPN must be disconnected for lab sessions. A WireGuard tunnel's reduced MTU
causes large transfers to stall while small ones succeed, which presents as a
mysterious hang during package installation rather than as a network error. Removing
the variable now means that any failure encountered later is genuinely a
configuration error.

## Step 2 — Confirm the VM datastore location

**Edit → Preferences → Workspace → Default location for virtual machines**

Set to the dedicated lab drive. The host OS drive has exceeded its rated write
endurance and must receive no virtual machine data.

Create the directory structure on the lab drive:

```
<lab-drive>\
├── ISO\          # Installation media
├── VMs\          # Virtual machines
└── Backups\      # Exported VMs
```

## Step 3 — Build the virtual machine

**Media:** **Ubuntu Server 26.04 LTS** (Resolute Raccoon, released 23 April 2026,
supported to April 2031) — download from `ubuntu.com/download/server`.

The correct file is named `ubuntu-26.04-live-server-amd64.iso` and is roughly 3 GB.
A file named `ubuntu-26.04-desktop-amd64.iso` at around 6 GB is the **wrong** edition:
it installs a full graphical desktop, which defeats the purpose of this phase.

Verify the SHA-256 checksum against the value published in `SHA256SUMS` on
`releases.ubuntu.com` before use. Verifying installation media is a routine security
control and this is the phase to build the habit.

```powershell
Get-FileHash .\ubuntu-26.04-live-server-amd64.iso -Algorithm SHA256
```

**Version note for later phases:** 26.04 LTS is three months old at the time of
writing. Security platforms deployed in Phase 4 publish supported operating system
matrices, and a brand-new LTS is often absent from them. The Phase 4 machine follows
the vendor's supported version, whatever that is at the time — it is not assumed to be
this one.

**Specification**

| Setting | Value | Reason |
|---|---|---|
| Guest OS | Ubuntu 64-bit | |
| vCPU | 2 | Adequate; the CPU is not the constraint |
| RAM | 4 GB | Within the 24 GB lab budget |
| Disk | 25 GB, single file, **not** pre-allocated | Space efficiency and faster snapshots |
| Network | **NAT (VMnet8)** | Package installation requires internet; the firewall does not exist yet |
| Installation | Minimal, **no desktop environment** | A GUI defeats the purpose of this phase |

Enable OpenSSH server during installation. Do not install additional snaps.

**Why NAT and not the lab segment:** the lab LAN has no gateway until a firewall is
built in Phase 2. Attaching this VM to VMnet2 now would produce a machine with no
route to anywhere. Sequence matters.

## Step 4 — Establish snapshot discipline

Immediately after a clean installation and first `apt update && apt upgrade`:

1. Take a snapshot named `clean-install-YYYYMMDD`.
2. Deliberately break the machine — for example, corrupt `/etc/fstab` and reboot.
3. Observe the failure. Read the error. Do not fix it.
4. Restore the snapshot.

**Why:** the confidence to break things is the single most valuable habit in a lab.
It is only available to someone who has actually tested the restore path, rather than
assuming it works.

Record the snapshot names and outcome in `evidence/phase-00/02-snapshot-test.md`.

## Step 5 — SSH key-based access from the host

Windows includes an OpenSSH client. From PowerShell:

```powershell
ssh-keygen -t ed25519 -C "lab-access"
type $env:USERPROFILE\.ssh\id_ed25519.pub
```

Append the public key to `~/.ssh/authorized_keys` on the VM, then disable password
authentication in `/etc/ssh/sshd_config`:

```
PasswordAuthentication no
PubkeyAuthentication yes
```

Reload the service and verify that key-based login succeeds and password login is
refused.

**Why this matters beyond convenience:** this is the first hardening control applied
in the lab, and the reasoning behind it — eliminating credential guessing as an attack
path — appears directly in the Security+ syllabus.

## Step 6 — Command line curriculum

Work through these in order. All are free and legitimate.

| Resource | Use |
|---|---|
| **OverTheWire — Bandit** (`overthewire.org/wargames/bandit`) | Levels 0-20. The single best structured shell practice available. Legal, intentionally built for this |
| **The Missing Semester of Your CS Education** (MIT, `missing-semester.mit.edu`) | Shell tooling, data wrangling, command-line environment lectures |
| **Linux Journey** (`linuxjourney.com`) | Gap filling on specific topics |
| `man` and `tldr` | Primary reference. Reaching for a search engine before the manual page is a habit worth breaking |

**Areas that must become automatic**

| Area | Commands |
|---|---|
| Navigation and files | `cd` `ls` `find` `cp` `mv` `rm` `mkdir` |
| Reading and filtering text | `cat` `less` `head` `tail` `grep` `cut` `sort` `uniq` `wc` |
| Pipes and redirection | `\|` `>` `>>` `2>` `tee` |
| Permissions and ownership | `chmod` `chown` `umask`, users and groups |
| Packages | `apt update` `apt install` `apt list --installed` |
| Services | `systemctl status/start/stop/enable` |
| Logs | `journalctl` with `-u` `--since` `-p` `-f` |
| Processes | `ps` `top` `kill` |
| Network inspection | `ip a` `ss -tulpn` `ping` `dig` |

## Exit criteria

- [ ] Virtual Network Editor corrected; host has no adapter on the lab segment
- [ ] VM datastore relocated to the lab drive
- [ ] Ubuntu Server LTS installed from checksum-verified media, no GUI
- [ ] Snapshot taken, machine deliberately broken, snapshot restored successfully
- [ ] SSH key-based authentication working; password authentication disabled
- [ ] OverTheWire Bandit levels 0-20 completed
- [ ] **Exit test passed** (below)
- [ ] This document updated with what actually happened, including what went wrong

## Exit test

Without notes, without a search engine, and without assistance:

> Find every failed login attempt on this system in the last 24 hours, count them,
> and list the source addresses ordered by number of attempts.

Solve it, then write the command and an explanation of each component into
`evidence/phase-00/03-exit-test.md`.

The explanation is the point. A command copied from elsewhere that produces the right
output is not a pass. Being able to state what each flag does, and why the pipeline is
ordered as it is, is a pass.

This exact task is a routine SOC Tier 1 activity. It is the smallest real piece of the
job, and it is a reasonable place to prove the foundation holds.

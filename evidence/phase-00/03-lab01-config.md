# Phase 0 — Evidence 03: LAB01 configuration

**Exit criterion 3, part 2 of 2:** the virtual machine is built to the specification in
[`../../docs/vm-inventory.md`](../../docs/vm-inventory.md).
**Date:** 2026-07-29

---

## Command (Git Bash)

```bash
grep -E "^(displayName|guestOS|memsize|numvcpus|ethernet0\.(connectionType|vnet)|scsi0:0\.fileName)" LAB01.vmx
```

## Output

```
displayName = "LAB01"
guestOS = "ubuntu-64"
numvcpus = "2"
memsize = "4096"
scsi0:0.fileName = "LAB01.vmdk"
ethernet0.connectionType = "custom"
ethernet0.vnet = "VMnet8"
```

## Verification against specification

| Field | Specified | Actual | Match |
|---|---|---|---|
| vCPU | 2 | 2 | yes |
| RAM | 4096 MB | 4096 MB | yes |
| Guest OS | `ubuntu-64` | `ubuntu-64` | yes |
| Disk | single file | `LAB01.vmdk` | yes |
| Network | NAT | VMnet8 | yes |

**Note on the network value.** The specification says NAT; the file says `custom` with
vnet VMnet8. These are equivalent on this host. VMnet8 *is* the NAT network, and the
NAT and DHCP services run on that switch regardless of how the adapter is labelled.
`custom` binds the adapter explicitly to VMnet8; `nat` binds it to whichever network
VMware designates as NAT. The explicit form is preferred here for consistency with
VMnet2 and VMnet3.

## Installed system

| Item | Value |
|---|---|
| Distribution | Ubuntu 26.04 LTS |
| Kernel | `7.0.0-28-generic x86_64` |
| Install type | Standard server, no desktop environment |
| Storage | Guided, whole disk, LVM |
| Hostname | `lab01` |
| SSH | OpenSSH server installed at setup, no keys imported |
| Updates | `apt update` and `apt full-upgrade` applied, reboot completed |
| Address on `ens32` | `192.168.231.129` — within the VMnet8 range, as designed |

**Criterion met.**

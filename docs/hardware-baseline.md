# Hardware Baseline

Host specification for this lab, recorded for reproducibility.

**Sanitisation notice:** this document deliberately omits host name, machine
identifiers, exact OS patch level, drive serial numbers and MAC addresses.
See [`sanitization-policy.md`](sanitization-policy.md) for the rules applied.

---

## 1. Host system

| Item       | Value                                                       |
| ---------- | ----------------------------------------------------------- |
| Platform   | AMD Socket AM5, X670 chipset (consumer ATX)                 |
| CPU        | AMD Ryzen 9 9900X — 12 cores / 24 threads, Zen 5, 120 W TDP |
| GPU        | NVIDIA GeForce RTX 3070 Ti, 8 GB                            |
| Host OS    | Windows 11 Pro 64-bit (25H2 branch)                         |
| Wired NIC  | Realtek Gaming 2.5GbE Family Controller                     |
| Wireless   | Disabled — antennas physically removed                      |
| Hypervisor | VMware Workstation Pro (free tier, no licence key required) |

The host is wired-only by design. Removing the wireless attack surface entirely is a
reasonable hardening choice for a machine that will run intentionally vulnerable
virtual machines.

## 2. Memory

| Item | Value |
|---|---|
| Installed | 32 GB (2 x 16 GB DDR5-6400) |
| Populated slots | 2 of 4 |
| Expansion headroom | 64 GB |

**Memory is the binding constraint of this lab, not CPU.** With ~8 GB reserved for the
Windows host, roughly 24 GB remains for virtual machines. Every phase in
`ROADMAP.md` is sized against a *concurrent VM* budget rather than a total VM count.

Populating all four AM5 slots typically forces a lower memory clock. For a
virtualisation host, capacity outweighs latency — this is an acceptable trade if the
lab later expands to 64 GB.

## 3. Storage

Four NVMe drives are present in the host. **One drive is dedicated to the lab.**

| Role | Class | Capacity available | Health |
|---|---|---|---|
| Lab drive | NVMe PCIe 4.0 x4, 1 TB | ~930 GB free, dedicated | 98 % |
| Host OS drive | NVMe PCIe 3.0 x4, 250 GB | — | 75 % — see risk below |
| Other drives | NVMe, 1 TB and 2 TB | — | 94 % / 100 % |

### Risk 1 — host OS drive endurance

The 250 GB boot drive carries a manufacturer endurance rating of 150 TBW. Recorded
host writes are approximately **196 TB, around 130 % of rated endurance**.
SMART-reported health has fallen to 75 %.

Mitigations applied to the lab design:
- No virtual machines, snapshots or ISO images are stored on the host OS drive.
- A full system image backup is a prerequisite before Phase 1.
- Drive replacement is a planned maintenance item, not an emergency.

### Risk 2 — lab drive space budget

The drive is dedicated to the lab and effectively empty: ~930 GB free of 931 GB.
This is comfortable, but snapshots are the item that consumes it. VMware snapshots
grow as delta files and can silently absorb tens of gigabytes per machine.

Planned allocation:

| Item | Estimated | Notes |
|---|---|---|
| ISO repository | 40 GB | Windows Server, Windows client, Linux, firewall |
| Firewall VM | 10 GB | |
| Linux VMs (2) | 40 GB | |
| Windows Server VM | 70 GB | |
| Windows client VM | 70 GB | |
| SIEM / log platform VM | 150 GB | Grows continuously with ingested logs |
| Snapshot headroom | 200 GB | The item most often underestimated |
| **Total** | **~580 GB** | Leaves ~350 GB margin |

**Operating rule:** snapshots are working tools, not backups. They are deleted once a
phase is verified and its state is documented. A drive that fills up mid-phase stops
the lab.

### Risk 3 — lab drive thermals

The lab drive was the warmest of the four at capture time (50 °C idle). Sustained VM
I/O will push this higher. NVMe thermal throttling typically begins around 70 °C.
Temperature should be re-checked under load before Phase 4, when the SIEM introduces
continuous writes.

## 4. Virtualisation platform notes

**Hypervisor-protected Code Integrity (HVCI) is enabled on the host.** HVCI depends on
Virtualization-Based Security, which keeps the Windows hypervisor active. Third-party
hypervisors then run on the Windows Hypervisor Platform rather than directly on the
CPU virtualisation extensions.

Practical consequences:
- Measurable performance overhead inside guests.
- Constraints on nested virtualisation.

Disabling HVCI improves lab performance and weakens host security posture. Because
this host will run intentionally vulnerable machines, the default position is to
**leave HVCI enabled** and accept the overhead. The 12-core CPU has ample headroom to
absorb it.

**Decision:** HVCI remains enabled. Revisit only if a specific phase requires nested
virtualisation.

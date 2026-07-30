# Virtual Machine Inventory

Target inventory, memory budget per phase, and the audit of the machines that
preceded this build.

---

## 1. Memory budget

| Item | Value |
|---|---|
| Physical memory | 32 GB |
| Reserved for host OS | 8 GB |
| **Available to virtual machines** | **24 GB** |

Every phase must fit inside 24 GB of *concurrently powered-on* machines. Exceeding it
does not produce an error — it produces swapping to disk, which on a lab drive means
degraded performance and additional wear on flash storage.

| Phase | Machines powered on | Total | Headroom |
|---|---|---|---|
| 0 | Linux workstation (4) | 4 GB | 20 GB |
| 1 | Linux workstation (4) + Windows client (4) | 8 GB | 16 GB |
| 2 | Firewall (2) + Linux (4) + Windows client (4) | 10 GB | 14 GB |
| 3 | Firewall (2) + Domain controller (4) + Windows client (4) | 10 GB | 14 GB |
| 4 | Firewall (2) + DC (4) + client (4) + SIEM (8) | 18 GB | 6 GB |
| 5 | Firewall (2) + DC (4) + client (4) + SIEM (6) + attacker (4) + target (1) | 21 GB | 3 GB |

**Phase 5 reaches the limit of this host.** Expansion to 64 GB is recommended before
Phase 5 begins, but is not required for Phases 0-4. That is several months of runway
in which to budget for it.

## 2. Target inventory

Machines are built **only when their phase unlocks**. Building early is not preparation;
it consumes disk, and on Microsoft evaluation media it starts a licence clock that
expires long before the machine is used.

| Role       | Guest OS             | vCPU | RAM    | Disk   | Network                | Phase |
| ---------- | -------------------- | ---- | ------ | ------ | ---------------------- | ----- |
| `LAB01`    | Ubuntu Server LTS    | 2    | 4 GB   | 25 GB  | NAT → later LAN        | 0     |
| `CLIENT01` | Windows 11           | 2    | 4 GB   | 60 GB  | LAN                    | 1     |
| `FW01`     | firewall appliance   | 1    | 2 GB   | 10 GB  | WAN + mgmt + LAN       | 2     |
| `DC01`     | Windows Server       | 2    | 4 GB   | 60 GB  | LAN                    | 3     |
| `SIEM01`   | Ubuntu Server LTS    | 2    | 6-8 GB | 150 GB | LAN                    | 4     |
| `ATK01`    | Kali Linux           | 2    | 4 GB   | 80 GB  | LAN                    | 5     |
| `TGT01`    | vulnerable appliance | 1    | 1 GB   | 8 GB   | **Detonation segment** | 5     |

### Evaluation licence timing

| Media | Evaluation period |
|---|---|
| Windows Server (Evaluation Center) | 180 days |
| Windows 11 Enterprise (Evaluation Center) | 90 days |

The clock starts at installation, not at first use. This is the concrete reason the
domain controller is built in Phase 3 and not earlier.

## 3. Audit of the previous build

Six machines existed from an earlier attempt, built by following a video tutorial
without understanding the configuration. All were decommissioned. Findings are
recorded here because the reasoning is the lesson, not the outcome.

| Machine | RAM | Network | Finding |
|---|---|---|---|
| Windows 11 client | 4 GB | VMnet2 | Sound configuration; wrong phase |
| Windows Server 2025 DC | 4 GB | VMnet2 | Evaluation clock started months before intended use |
| Kali attacker | 8 GB | VMnet2 | Over-provisioned; 6 vCPU unnecessary in a lab |
| Ubuntu blue team | 6 GB | VMnet2 | Wrong phase; no log sources existed to collect from |
| Firewall | 1 GB | VMnet8 / VMnet1 / VMnet2 | Interface mapping correct; internal configuration unverifiable |
| **Vulnerable target** | 512 MB | **VMnet2** | **Critical — see below** |

### Critical finding: vulnerable target on a host-reachable segment

An intentionally vulnerable machine — one distributed specifically because it ships
with unauthenticated remote root access and backdoored network services — was attached
to VMnet2.

At the time, VMnet2 was a host-only network **with the host virtual adapter enabled**.
Two consequences followed:

1. The vulnerable machine had a direct network path to the Windows host.
2. Routed through the firewall's WAN interface on the NAT network, it also had
   outbound internet access.

Neither is acceptable. A machine of that class belongs on a segment with no host
adapter, no route to the internet, and a default-deny policy — the detonation segment
defined in [`lab-architecture.md`](lab-architecture.md).

**This is the single most important lesson from the previous attempt:** the tutorial
produced a lab that looked correct and worked. It was not safe, and there was no way
to know that without understanding what each setting did.

### Decommissioning rationale

The firewall's interface mapping was correct and could arguably have been retained.
It was deleted anyway, for one reason: its internal configuration — firewall rules,
DHCP scope, administrative credentials — could not be audited by someone who did not
write it.

A security appliance that cannot be audited is not a security boundary. It is an
assumption. Rebuilding it costs roughly thirty minutes and produces something whose
every rule has a documented reason.

## 4. Naming convention

| Prefix | Meaning |
|---|---|
| `FW` | Firewall / gateway |
| `DC` | Domain controller |
| `SIEM` | Log collection and analysis |
| `CLIENT` | Endpoint under monitoring |
| `ATK` | Offensive tooling |
| `TGT` | Intentionally vulnerable target |
| `LAB` | General purpose workstation |

Role-based names keep the documentation publishable and make log correlation readable.
A host name that describes a function is worth more in a SIEM than one that describes
a person.

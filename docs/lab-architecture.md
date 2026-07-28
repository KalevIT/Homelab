# Lab Architecture

Network design for the virtual lab, with the reasoning behind each decision.

> **Status:** virtual network configuration verified on the host (2026-07-28).
> Firewall design, addressing plan and policy are unbuilt and unverified — they become
> real in Phase 2, not before. Nothing below section 2 should be treated as working.

---

## 1. Design principle

Three separation levels, each with a different trust assumption:

1. **Uplink** — the only path to the real internet. Terminates at the firewall, never
   at a lab machine.
2. **Management** — the only network the Windows host can reach. Exists so the host
   can administer the firewall without touching the lab.
3. **Lab** — where machines live. The Windows host has **no interface here at all**.

The rule that makes this design safe is simple: *the segment where dangerous things
happen must not be reachable from the host operating system.*

## 2. Virtual network configuration

| Network | VMware type | Host virtual adapter | VMware DHCP | Subnet | Role |
|---|---|---|---|---|---|
| VMnet0 | Bridged | n/a | n/a | Host LAN | **Never used by any lab machine** |
| VMnet8 | NAT | Enabled (default) | Enabled (default) | 192.168.231.0/24 | Firewall WAN → internet |
| VMnet1 | Host-only | **Enabled** | **Enabled** | 192.168.31.0/24 | Firewall management |
| VMnet2 | Host-only | **Disabled** | **Disabled** | 10.10.10.0/24 | Lab LAN |
| VMnet3 | Host-only | **Disabled** | **Disabled** | 10.10.20.0/24 | Detonation segment (Phase 5) |

VMnet1 and VMnet8 subnets are VMware-assigned defaults on this host and are recorded
here as observed, not chosen.

### VMnet0 (bridged) is permanently out of scope

A bridged adapter places a virtual machine directly on the physical home network,
alongside every other device in the household. No lab machine is ever attached to it.
Tutorials frequently recommend bridged networking "for simplicity"; in a lab that will
host intentionally vulnerable systems, it removes every boundary this design exists to
create.

Both settings are configured per network in **Edit → Virtual Network Editor**
(administrator privileges required):

- `Connect a host virtual adapter to this network`
- `Use local DHCP service to distribute IP addresses to VMs`

### Why the host virtual adapter must be disabled on VMnet2 and VMnet3

A VMware host-only network is **not isolated by default**. Broadcom's documentation is
explicit: host-only networking creates a virtual Ethernet adapter that is *visible to
the host operating system*, connecting the virtual machines and the host to a shared
private network.

Consequently, a segment labelled "isolated" while the host virtual adapter is enabled
gives every machine on it a direct network path to the Windows host. Once
intentionally vulnerable machines or live malware samples are introduced, that path
becomes the escape route.

Clearing the checkbox removes the host's interface from the switch. The virtual
machines continue to communicate with each other; the host simply is not on the
network any more.

*Alternative:* VMware **LAN Segments** (configured per virtual machine, not in the
Virtual Network Editor) provide fully private switches with no host adapter and no
DHCP. They are not limited to the 20-VMnet ceiling and are the cleaner option once
the lab needs more than a handful of segments.

### Why VMware DHCP must be disabled on VMnet2 — but not on VMnet1

VMware enables its own DHCP service on host-only networks by default.

**On VMnet2 it must be disabled.** The firewall will serve DHCP on the lab LAN. Two
DHCP servers in the same broadcast domain race each other: clients receive addresses,
gateways and DNS servers from whichever responds first, and the symptom is
intermittent and unpleasant to diagnose. The firewall must be the single source of
addressing on the lab LAN — that is the entire point of putting a firewall in the lab.

**On VMnet1 it must stay enabled.** The host's own VMnet1 adapter obtains its address
from VMware's DHCP service. Disabling it leaves the host adapter with a link-local
address and no path to the firewall management interface. There is no conflict on this
segment: the firewall's management interface is statically addressed and serves no
DHCP there.

*Alternative:* VMware DHCP on VMnet1 may be disabled if the host's VMnet1 adapter is
given a static address manually in Windows. That is more deterministic, and more steps.
The simpler option is chosen here deliberately — complexity budget belongs on the lab
LAN, not on the management path.

### Why management sits on its own network

Keeping administration on VMnet1 is what allows VMnet2 to have no host adapter at
all. Without a separate management path, the host would need an interface on the lab
LAN to reach the firewall web interface — which is exactly the thing being avoided.

This is out-of-band management, and it is the same pattern used in production
networks.

## 3. Firewall interface assignment

| Interface | Network | Address | Purpose |
|---|---|---|---|
| WAN | VMnet8 | DHCP from VMware NAT | Internet uplink |
| LAN | VMnet2 | 10.10.10.1/24 | Lab gateway, DHCP server, DNS resolver |
| OPT1 | VMnet1 | 192.168.31.10/24 **(static)** | Management interface |

The management address is static and chosen outside VMware's DHCP pool on VMnet1.

## 4. Baseline firewall policy

Written before the rules are created, so that each rule has a stated purpose.

| Source | Destination | Action | Reason |
|---|---|---|---|
| Management (OPT1) | Firewall web interface | Allow | Administration path |
| Management (OPT1) | LAN | Allow | Troubleshooting from host |
| LAN | WAN | Allow | Updates and package installation |
| LAN | Management (OPT1) | **Block** | Lab machines must never reach the host |
| LAN | Firewall web interface | **Block** | A compromised lab machine must not administer the gateway |
| Detonation (Phase 5) | Anything | **Block** | Default-deny; exceptions added explicitly and documented |

Every rule added later must be logged in `CHANGELOG.md` with its justification. A
firewall rule with no recorded reason is a rule nobody can safely remove.

## 5. Addressing plan

| Range | Assignment |
|---|---|
| 10.10.10.1 | Firewall LAN interface / gateway |
| 10.10.10.2 – 10.10.10.49 | Static — infrastructure (domain controller, SIEM) |
| 10.10.10.50 – 10.10.10.99 | Static — clients under test |
| 10.10.10.100 – 10.10.10.200 | DHCP pool served by the firewall |
| 10.10.10.201 – 10.10.10.254 | Reserved |

Infrastructure receives static addresses because log sources are identified by IP.
A domain controller that changes address breaks every correlation rule built on top
of it.

## 6. Host-side interference

Two host conditions affect lab networking and must be known before troubleshooting
anything inside a virtual machine.

### Commercial VPN client

A WireGuard-based VPN client is installed on the host and may be connected. When it
holds the default route, VMware NAT traffic egresses through the tunnel.

Consequences:
- WireGuard uses a reduced MTU (typically 1420 bytes). Traffic that fits a standard
  1500-byte Ethernet frame does not fit the tunnel. Small requests succeed and large
  transfers stall — package installation hanging partway through is the classic
  symptom, and it does not look like a network problem.
- Some VPN kill-switch implementations block virtual adapter traffic outright.

**Working rule: disconnect the VPN during lab sessions, at least through Phase 2.**
The objective in the early phases is to be certain that a networking failure is a
configuration error rather than an external variable. Reintroduce it later, knowingly,
once the baseline is understood.

### Filtering DNS resolver

The host resolves through a filtering DNS service that blocks malware and adult
content categories. VMware NAT passes host DNS settings to guests by default.

This is harmless through Phase 3. From Phase 4 onward the lab firewall becomes the
resolver for the lab LAN, which removes the dependency. In Phase 5 a filtering
resolver would silently interfere with attack simulation traffic — noted here so that
it is remembered rather than rediscovered.

## 7. Verification checklist

Each item is confirmed with evidence stored under `evidence/`, in the folder of the
phase that proves it:

- [x] Windows host has **no** network adapter on the lab subnet (`ipconfig /all`)
      — verified 2026-07-28, `evidence/phase-00/01-host-adapters.txt`
- [ ] Windows host **can** reach the firewall management address
- [ ] A lab VM receives its address from the firewall, not from VMware
- [ ] A lab VM **cannot** reach the Windows host
- [ ] A lab VM can reach the internet through the firewall
- [ ] Blocked traffic appears in the firewall log

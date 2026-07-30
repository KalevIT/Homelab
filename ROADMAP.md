# Roadmap

Phases unlock sequentially. A phase is complete only when **every** exit criterion is
met and the corresponding document exists in `docs/phases/`.

Starting point: LPI Linux Essentials achieved (theory), limited hands-on shell
experience. The early phases are deliberately narrow. Building a full SIEM lab before
knowing how to read a log file is the most common way home labs are abandoned.

## Effort model

Available time: approximately **one hour per day**. Planning assumes **5 effective
hours per week**, not 7 — sessions are missed, and a plan that assumes perfect
attendance is a plan that fails in week three.

| Phase | Estimated effort | Elapsed at 5 h/week |
|---|---|---|
| 0 — Foundations | 12-15 h | ~3 weeks |
| 1 — Networking + client | 15-18 h | ~3-4 weeks |
| 2 — Segmentation | ~20 h | ~4 weeks |
| 3 — Active Directory | ~25 h | ~5 weeks |
| 4 — Log collection and SIEM | ~35 h | ~7 weeks |
| 5 — Detection engineering | ~40 h | ~8 weeks |
| **Total** | **~150 h** | **~30 weeks** |

These are estimates, not commitments. They will be corrected against actual recorded
time as each phase completes — that correction is more valuable than the original
estimate.

## Session structure

At one hour per session, the dominant cost is not the work. It is re-establishing
context at the start and losing it at the end.

| Time | Activity |
|---|---|
| 5 min | Read the last session note. Do not start anything before this |
| 45 min | **One** defined objective. Not two |
| 10 min | Write what happened, what broke, and what comes next |

The final ten minutes are not overhead. They *are* the documentation this repository
exists to hold. Written at the point of doing, documentation costs ten minutes;
reconstructed weeks later, it costs an evening and is less accurate.

---

## Phase 0 — Linux and CLI foundations

**Concurrent VMs:** 1 | **RAM budget:** 4 GB | **Status:** In progress

Full build sheet: [`docs/phases/phase-00-foundations.md`](docs/phases/phase-00-foundations.md)

Build a single Linux virtual machine and become genuinely comfortable in a terminal.
No security tooling. No attack tools. This phase exists because every later phase
assumes the shell is not an obstacle.

**Exit criteria**
- [x] Virtual Network Editor corrected; host has no adapter on the lab segment
- [x] VM datastore relocated to the dedicated lab drive
- [x] Ubuntu Server LTS installed from checksum-verified media, no GUI
- [x] Snapshot taken, VM deliberately broken, snapshot restored successfully
- [x] SSH key-based authentication working; password authentication disabled
- [ ] OverTheWire *Bandit* levels 0-20 completed
- [ ] Exit test passed and explained in writing
- [ ] `phase-00-foundations.md` updated with what actually happened

---

## Phase 1 — Networking fundamentals + Windows client

**Concurrent VMs:** 2 | **RAM budget:** 8 GB | **Status:** Locked

**Exit criteria**
- [ ] Windows 10/11 evaluation VM built and joined to an isolated virtual network
- [ ] Static IP addressing plan documented in `docs/lab-architecture.md`
- [ ] Two VMs communicate on an isolated network with no internet access
- [ ] Traffic between the two VMs captured and read in Wireshark
- [ ] Can explain, in writing: TCP three-way handshake, DNS resolution, DHCP lease,
      ARP, NAT, and the difference between a port and a protocol
- [ ] Host system image backup completed (mitigates the `C:` endurance risk)

---

## Phase 2 — Network segmentation

**Concurrent VMs:** 3 | **RAM budget:** 12 GB | **Status:** Locked

**Exit criteria**
- [ ] Firewall appliance (OPNsense or pfSense) deployed as lab gateway
- [ ] At least two segmented networks with an enforced traffic policy between them
- [ ] Firewall rules documented with the reason for each rule
- [ ] Blocked traffic demonstrated and captured in firewall logs

---

## Phase 3 — Active Directory domain

**Concurrent VMs:** 3 | **RAM budget:** 14 GB | **Status:** Locked

**Exit criteria**
- [ ] Windows Server domain controller promoted
- [ ] Client machine domain-joined
- [ ] Organisational units, users and groups created to a documented design
- [ ] At least one Group Policy Object created and its effect verified on the client

---

## Phase 4 — Log collection and SIEM

**Concurrent VMs:** 3-4 | **RAM budget:** 20 GB | **Status:** Locked

This is the phase that will expose the 32 GB memory limit. Platform choice
(lightweight agent-based vs. full monitoring stack) depends on available RAM at the
time and must be re-evaluated, not assumed now.

**Exit criteria**
- [ ] Central log platform deployed
- [ ] Windows and Linux endpoints forwarding logs successfully
- [ ] Sysmon deployed on Windows with a documented configuration
- [ ] A dashboard exists that answers one specific operational question

---

## Phase 5 — Detection engineering

**Concurrent VMs:** 4 | **RAM budget:** 22 GB | **Status:** Locked

**Exit criteria**
- [ ] Deliberately vulnerable target machine deployed on an isolated segment
- [ ] Three attack techniques executed and mapped to MITRE ATT&CK
- [ ] A working detection rule written for each technique
- [ ] Three incident reports written in the format used by a real SOC

---

## Progress tracking

| Phase | Exit criteria met | Total |
|---|---|---|
| 0 | 5 | 8 |
| 1 | 0 | 6 |
| 2 | 0 | 4 |
| 3 | 0 | 4 |
| 4 | 0 | 4 |
| 5 | 0 | 4 |

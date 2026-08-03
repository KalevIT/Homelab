# Home Security Lab

A documented, phase-based home cybersecurity laboratory built on a single workstation.
Purpose: build practical blue-team skills and support CompTIA Security+ (SY0-701) preparation.

> **Status:** Phase 0 — Foundations (in progress)
> **Last updated:** 2026-08-03

---

## 1. Why this repository exists

This is not a collection of tutorials. It is a build log.
Every phase records: what was built, why, how it was verified, and what broke.

Four rules for this repo:

1. **Nothing is documented before it works.** Documentation follows a verified result.
2. **Every claim has a source.** Vendor docs, RFCs, or official project documentation — linked.
3. **Evidence over description.** Screenshots, config exports and command output live in `evidence/`.
4. **Sanitise before committing.** See [`docs/sanitization-policy.md`](docs/sanitization-policy.md).

### File conventions

| Rule | Detail |
|---|---|
| Every document is Markdown | `.md` only, with no exceptions, so that one convention covers the whole repository |
| Verbatim output goes in fenced blocks | Command output is wrapped in a fenced code block and never edited to render better |
| Evidence is numbered per phase | `evidence/phase-NN/NN-short-name.md`, numbered in the order produced |
| English throughout | Documentation is written in English regardless of working language |

Wrapping raw output in a fenced block is what makes `.md` safe for evidence: Markdown
would otherwise interpret characters that appear routinely in command output —
asterisks, underscores, hashes — as formatting instructions.

---

## 2. Hardware baseline

Full detail: [`docs/hardware-baseline.md`](docs/hardware-baseline.md)

| Component | Spec |
|---|---|
| CPU | AMD Ryzen 9 9900X — 12C / 24T (Zen 5) |
| RAM | 32 GB DDR5-6400 (2 of 4 slots populated) |
| Platform | AMD Socket AM5, X670 chipset |
| Host OS | Windows 11 Pro (25H2 branch) |
| Lab storage | 1x NVMe PCIe 4.0, ~930 GB dedicated |
| Networking | Wired 2.5GbE only; wireless physically removed |
| Hypervisor | VMware Workstation Pro (free tier) |

**Primary constraint: RAM, not CPU.** The CPU can host far more virtual machines than
32 GB of memory allows. Lab phases are therefore designed around a *concurrent VM budget*,
not around total VM count.

---

## 3. Repository structure

```
.
├── README.md                  # This file — entry point and current status
├── ROADMAP.md                 # Phases, exit criteria, progress tracking
├── CHANGELOG.md               # Dated log of what actually changed
├── docs/
│   ├── hardware-baseline.md   # Verified host specs and known constraints
│   ├── lab-architecture.md    # Network design, IP plan, firewall policy
│   ├── sanitization-policy.md # What may and may not be published here
│   ├── vm-inventory.md        # Target machines, memory budget, previous-build audit
│   ├── phases/                # One document per phase, written as it is completed
│   └── exercises/             # Reusable practice material written during a phase
├── vm-configs/                # Exported VM settings, .vmx notes, build sheets
├── scripts/                   # Automation (PowerShell / Bash), one folder per purpose
└── evidence/                  # Command output and config dumps, one folder per phase
    └── phase-00/              # 01-host-adapters.md, 02-iso-checksum.md, ...
```

**Why this layout:** `docs/` is human-readable narrative, `evidence/` is raw proof,
`vm-configs/` and `scripts/` are reproducible artefacts. A reader can verify any claim
in `docs/` by opening the matching folder in `evidence/`.

---

## 4. Phases

| Phase | Focus | Concurrent VMs | Status |
|---|---|---|---|
| 0 | Linux and CLI foundations | 1 | In progress |
| 1 | Networking fundamentals + Windows client | 2 | Locked |
| 2 | Network segmentation (firewall / VLANs) | 3 | Locked |
| 3 | Active Directory domain | 3 | Locked |
| 4 | Log collection and SIEM | 3-4 | Locked |
| 5 | Detection engineering and attack simulation | 4 | Locked |

Phases unlock sequentially. Exit criteria are defined in [`ROADMAP.md`](ROADMAP.md).

---

## 5. Prior build

Six virtual machines from an earlier attempt were audited and decommissioned before
this repository was created. The audit — including a vulnerable target found on a
host-reachable segment — is documented in [`docs/vm-inventory.md`](docs/vm-inventory.md).

The reasoning is kept deliberately. A lab that works is not the same as a lab that is
safe, and the difference is only visible to someone who understands the configuration.

## 6. Scope and ethics

All activity is confined to isolated virtual networks under the operator's ownership.
No testing is performed against systems or networks belonging to third parties.
Offensive tooling is used exclusively against intentionally vulnerable machines built
inside this lab.

---

## 7. Certification track

| Certification | Status | Target |
|---|---|---|
| Cambridge English B2 First (FCE) | Achieved | Dec 2025 |
| LPI Linux Essentials | Achieved | Apr 2026 |
| CompTIA Security+ SY0-701 | Not started | TBD |
| CompTIA CySA+ | Not started | TBD |

Phases 0-2 are lab only. Certification study runs alongside the lab from Phase 3
onward, where the material and the build reinforce each other.

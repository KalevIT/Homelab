# Changelog

## 2026-07-28 (2) — Repository published

### Fixed
- `.gitignore` had been created without its leading dot, because Windows Explorer
  silently strips it. The file was inert and Obsidian workspace metadata was
  committed. Renamed via PowerShell and untracked with `git rm -r --cached`.

### Note
- `git rm --cached` removes files from future commits only. The initial commit still
  contains them. Harmless here; had it been a credential, the only correct response
  would have been revocation and a fresh repository.

## 2026-07-28 (1) — Clean slate verified, host findings recorded

### Done
- All six previous virtual machines deleted from disk. Lab drive now ~930 GB free of
  931 GB. The earlier 650 GB figure predated the deletion.
- Virtual networks reset to VMware defaults. VMnet2 no longer exists and must be
  created rather than corrected.
- Repository created locally with no README, no gitignore and no licence, so the
  prepared files can be committed as the initial state.

### Recorded
- Observed subnets on this host: VMnet1 `192.168.31.0/24`, VMnet8 `192.168.231.0/24`.
  Documentation updated from the placeholder values used previously.
- VMnet0 (bridged) declared permanently out of scope for lab machines.
- Host runs a WireGuard-based commercial VPN client. Reduced tunnel MTU stalls large
  transfers while small ones succeed, which presents as an unexplained hang rather than
  a network error. VPN is to be disconnected during lab sessions through Phase 2.
- Host resolves through a filtering DNS service. Harmless until Phase 4; would
  silently interfere with Phase 5 traffic.

### Corrected
- **Previous instruction was wrong.** VMware DHCP was to be disabled on VMnet1 as well
  as VMnet2. The host's own VMnet1 adapter obtains its address from that service, so
  disabling it would leave the host with a link-local address and no route to the
  firewall management interface. VMnet1 DHCP stays enabled; there is no competing DHCP
  server on that segment.
- Phase 0 media corrected to **Ubuntu Server 26.04 LTS**
  (`ubuntu-26.04-live-server-amd64.iso`, ~3 GB). A desktop ISO had been downloaded.

## 2026-07-27 (4) — Previous build audited and decommissioned

### Added
- `docs/vm-inventory.md` — target machine list, per-phase memory budget, evaluation
  licence timing, and the audit of the six machines from the previous attempt.
- `LICENSE` (MIT).

### Removed
- All six virtual machines from the previous build, deleted from disk.

### Critical finding
- An intentionally vulnerable target machine was attached to VMnet2 while that network
  had the host virtual adapter enabled, giving it a direct path to the Windows host and
  outbound internet access through the firewall. This is the finding that justifies the
  network correction recorded in the previous entry.

### Rationale for full rebuild
- The firewall's interface mapping was correct, but its internal configuration could
  not be audited by someone who did not write it. A security appliance that cannot be
  audited is an assumption, not a boundary.
- Machines are now built only when their phase unlocks. Building early consumes disk
  and starts Microsoft evaluation licence clocks months before the machine is used.

## 2026-07-27 (3) — Phase 0 opened

### Added
- `docs/phases/phase-00-foundations.md` — Phase 0 build sheet: network correction,
  datastore relocation, Ubuntu Server build, snapshot discipline, SSH hardening,
  command line curriculum and a written exit test.

### Changed
- Phase 0 duration reduced from ~4 weeks to ~2 weeks. LPI Linux Essentials supplies
  the concepts; this phase converts them into practice rather than teaching them.
- Storage decision confirmed: the 2 TB drive is unavailable. The entire lab runs on a
  single 1 TB drive with ~650 GB free. Snapshot hygiene is therefore mandatory, not
  optional.

## 2026-07-27 (2) — Design correction and sanitisation

### Added
- `docs/lab-architecture.md` — virtual network design, firewall interface assignment,
  baseline policy, addressing plan and a verification checklist.
- `docs/sanitization-policy.md` — rules for what may be published in this public
  repository.

### Changed
- `docs/hardware-baseline.md` rewritten to remove host name, machine identifiers,
  exact OS revision and drive-level detail.
- Storage plan revised: a single dedicated 1 TB NVMe drive (~650 GB free) hosts the
  entire lab, including ISO images and snapshots. Space budget documented.
- HVCI decision recorded: remains **enabled**. The host will run intentionally
  vulnerable machines; the performance cost is acceptable on a 12-core CPU.

### Corrected
- Existing VMnet2, described as an isolated lab network, was configured as a
  host-only network with the host virtual adapter attached. Host-only networking in
  VMware connects the host to the segment by design, so the network was not isolated.
  Design now requires the host virtual adapter to be **cleared** on all lab segments.
- VMware's local DHCP service must be **disabled** on segments where the firewall
  serves DHCP, to avoid two DHCP servers in one broadcast domain.

## 2026-07-27 (1) — Repository initialised

### Added
- Documentation-first repository structure.
- `docs/hardware-baseline.md` — verified host specification.
- `ROADMAP.md` — six phases with explicit exit criteria.

### Identified risks
- Host OS drive has exceeded its rated 150 TBW endurance (~196 TB written, SMART
  health 75 %). Excluded from all lab storage.
- 32 GB system memory is the binding constraint on concurrent virtual machines.
- Lab drive runs warmest of the four; thermals to be re-checked under load.

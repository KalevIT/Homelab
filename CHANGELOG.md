# Changelog

## 2026-08-03 — Consolidation completed; exit test attempted, not passed

### Done
- Consolidation sessions F, G and H completed: permissions and privilege, change
  detection by comparison, network service discovery.

### Added
- `evidence/phase-00/08-startup-baseline.md` — hashes and file listing for the shell
  startup files, recorded while `.bashrc` was still byte-identical to the distribution
  template. The value of a baseline is set by when it was taken.
- `docs/exercises/log-analysis.md` — a session covering `journalctl`, event counting
  and pipeline ordering.

### Exit test: attempted, not passed
- The test requires `journalctl`, which appears in the Phase 0 curriculum but had
  never been practised. Every other tool in this phase was exercised before being
  required; this one was not. The attempt therefore measured the sequencing rather
  than the understanding, and the criterion stays open.
- Recorded as a prerequisite in `phase-00-foundations.md`: the consolidation sessions
  come first, then the test.

### Findings from the attempt
- **Counting log lines is not counting events.** One failed SSH login produces two
  journal entries sharing a PID, both containing the phrase being searched for. A
  case-insensitive match counted eleven where there were six. The number was
  plausible, which is why the error survives review.
- **A pipeline returned the right answer for the wrong reason.** `sort` had been
  placed before `cut`, so it ordered full log lines by timestamp rather than grouping
  addresses. It agreed with the correct version only because each address happened to
  occupy a contiguous block of time — a property of the data, not of the command.
  Interleaved sources would have miscounted.
- The second finding was raised by asking whether a correct result might be specific
  to the sample. Suspecting a right answer is the harder half of verification.

### Principles recorded
- A command that returns the right answer is not the same as a correct command.
- Prefer a native filter (`-u ssh`) to a text match: a text match is a guess about
  wording.
- `-i` is not free. It widens a filter, which is required in some contexts and
  harmful in others.
- Stating the boundary of an analysis is part of the analysis. "Six failed logins"
  invites the reading that there were no others.

## 2026-08-02 (2) — Criterion 5 was recorded as met before it was

### Fixed
- **Password authentication had remained enabled since 2026-07-30**, while
  `evidence/phase-00/05-ssh-key-auth.md` recorded the criterion as satisfied.
- Ubuntu's `sshd_config` includes `/etc/ssh/sshd_config.d/*.conf` at line 24; the
  manual edit sat at line 78. Cloud-init had written `PasswordAuthentication yes` into
  `50-cloud-init.conf` at install time, and for most SSH options the first value read
  wins. The included file was read first and the later edit was ignored. No error was
  produced.
- Corrected in the drop-in file, validated with `sshd -t`, confirmed with `sshd -T`,
  applied with a restart while holding the existing session open.

### Root cause
- The original verification tested that key-based login succeeded without a prompt, and
  concluded from that that passwords were disabled. The conclusion did not follow: SSH
  stops at the first authentication method that succeeds, so a working key hides
  whether a password would also have been accepted.
- Surfaced incidentally. A permissions exercise set the home directory to mode 777;
  `sshd` correctly refused the key and **fell back to a password prompt that
  succeeded**. The fallback should not have existed.

### Changed
- `05-ssh-key-auth.md` now records both required tests: that the key works, and that
  `ssh -o PubkeyAuthentication=no` is refused with `Permission denied (publickey)`.
- The document opens with a note that it was wrong when first written. The reasoning
  error is more useful than the corrected result.

### Principles recorded
- `sshd -T` reports the configuration in force; the file reports an intention.
- A control is not verified until its failure path has been tested.
- A setting believed applied but silently overridden is worse than one never attempted:
  it produces confidence without protection, and nothing signals the gap until
  something exercises it.

## 2026-08-02 — Bandit completed, Phase 0 at six of eight

### Done
- OverTheWire *Bandit* levels 0-20 completed. Phase 0 exit criterion 6 met.
- Level 20 onward is outside the scope defined in `ROADMAP.md` and was not attempted.

### Added
- `evidence/phase-00/06-bandit.md` — what each group of levels taught, with solutions
  and passwords deliberately excluded per OverTheWire's stated request.
- `evidence/phase-00/07-setuid-baseline.md` — the set of setuid binaries present on
  `LAB01` in a known-good state, with the comparison procedure.
- `docs/exercises/permissions-and-services.md` — three further consolidation sessions
  covering permissions and setuid, change detection by comparison, and network
  service discovery.

### Concepts recorded
- **setuid** grants the owner's privileges to any caller with execute permission. Not
  `sudo`: no prompt, no audit record. It is the mechanism behind most local privilege
  escalation, and the defensive counterpart is a recorded baseline.
- **Shell startup files are a persistence mechanism.** A file that executes on every
  login restarts anything written into it. Detection is a comparison problem: the
  modified file is indistinguishable from normal until placed beside a known-good copy.
- **Automated service identification is a hypothesis.** A scanner reports what a
  service claimed when probed. Confirming by hand is a separate step.
- **Encoding is not encryption.** No key, reversible by anyone. Distinct from
  encryption and from hashing.

### Method note
- Two early levels were solved by inspecting files individually rather than using the
  intended tool. This works at a scale of ten files and will not work in Phase 4.
  Recorded rather than tidied away.

## 2026-08-01 — Phase 0 paused for consolidation

### Added
- `docs/exercises/cli-fundamentals.md` — five sessions of practice covering output
  channels, file identification, text transformation, compression and network
  services, worked on the lab machine rather than on the wargame.

### Decision
- Bandit progress halted at level 15 of 21. Seven new tools had been used across seven
  consecutive levels, each introduced by a puzzle that had to be solved immediately.
  Every level was completed, which is not the same as having learned seven tools.
- The exercises exist to break that loop: same concepts, own machine, mistakes free.
  Several exercises induce a failure deliberately, because an observed failure is
  understood better than a described one.

### Note on scope
- Bandit level numbers are not mapped to exercises. OverTheWire asks that solutions not
  be published; naming a tool is harmless, since the level pages list the tools
  themselves, but publishing the arguments that solve a level is not.

## 2026-07-30 — SSH key authentication, passwords disabled

### Done
- Ed25519 key pair generated on the host, public key installed on `LAB01`.
- `PasswordAuthentication no` applied in `sshd_config`, validated with `sshd -t` before
  restart, tested from a second session while the first was held open.
- Phase 0 exit criterion 5 met. Five of eight.
- Evidence: `evidence/phase-00/05-ssh-key-auth.md`.

### Fixed
- `.gitignore` matched `id_rsa*` only — a pattern predating Ed25519. The key generated
  here is named `id_ed25519` and would not have been excluded. Patterns extended to
  `id_ed25519*`, `id_ecdsa*`, `*.pub`, `known_hosts*` and `authorized_keys`.
- Consequence had it gone unnoticed: a private key copied into the working tree would
  have been committed silently to a now-public repository.

### Recorded
- The key passphrase is intentionally empty. `LAB01` is an internal laboratory host and
  the risk is accepted. Standing rule adopted: any key reaching anything outside the
  lab carries a passphrase, with `ssh-agent` handling repetition.

## 2026-07-29 (2) — Single file convention adopted

### Changed
- All evidence files converted from `.txt` to `.md`. The repository now uses one
  document format throughout, with no exceptions.
- Verbatim command output is wrapped in fenced code blocks. This is what makes
  Markdown safe for evidence: without fencing, characters that appear routinely in
  command output — asterisks, underscores, hashes — would be interpreted as
  formatting rather than displayed.

### Added
- `README.md`: a **File conventions** section recording the rule, so the choice is
  documented rather than remembered.

### Rationale
- The earlier split (`.txt` for verbatim capture, `.md` for written documents) was
  defensible but imposed a second convention to keep track of for no functional gain
  once output is fenced. A single rule that is always true is easier to follow than a
  correct rule with exceptions.

### Fixed
- Evidence numbering made contiguous across the phase. Two Phase 0 steps referenced
  file numbers that collided with files already written.

## 2026-07-29 — LAB01 built, snapshot restore proven

### Done
- `LAB01` built and installed: Ubuntu Server 26.04 LTS, standard install, no desktop,
  2 vCPU, 4 GB, 25 GB LVM, attached to VMnet8. Media checksum verified before use.
- Snapshot `clean-install-20260729` created, `/etc/fstab` deliberately corrupted,
  boot failure into emergency mode observed, snapshot restored and verified clean.
- Four evidence files committed under `evidence/phase-00/`.
- Phase 0 exit criteria 1 to 4 met. Four of eight.

### Fixed
- Host OS recorded as Windows 11 Pro **25H2**, not 24H2. Build 26200 is 25H2; 24H2 is
  build 26100. The error was in the documentation, not the host, and was found by
  checking a figure supplied rather than accepting it.
- Documented datastore layout corrected to the structure actually built
  (`ISO\`, `DISKS\`, `GITHUB.REPO\`). The build was aligned to reality rather than
  the reverse.

### Corrected
- Phase 0 previously specified a **minimized** server install. This was wrong: the
  minimized variant removes manual pages, documentation and editors, and this phase
  relies on `man` as its primary reference. Standard "Ubuntu Server" is now specified.

### Recorded
- SHA-256 verification proves integrity, not authenticity. GPG verification of
  `SHA256SUMS` against Canonical's signing key was not performed and is logged as a
  known gap in `evidence/phase-00/02-iso-checksum.md`.
- The installer's LVM layout leaves part of the volume group unallocated by design
  (~11.5 GB of 23 GB assigned to `/`). Noted so it is not mistaken for a fault.

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

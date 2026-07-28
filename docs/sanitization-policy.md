# Sanitisation Policy

Rules applied to every file committed to this public repository.

The concern is not that the hardware is secret. Component models are shared by
hundreds of thousands of people and disclose nothing useful about this specific host.
The concern is **correlation and targeting**: identifiers that link this repository to
one machine, and configuration detail that tells an attacker what to try.

---

## 1. Never committed

| Category | Examples | Why |
|---|---|---|
| Host identifiers | Machine name, machine GUID, Windows product ID | Uniquely identifies this system across any other data source |
| Hardware serials | Drive serial numbers, board serial, UUIDs | Same — permanent, unique, unchangeable |
| Network identifiers | MAC addresses, public IP, ISP-assigned addressing, home LAN subnet, SSIDs | Locates and identifies the physical network |
| Exact patch level | Full Windows build with revision, precise driver versions | Discloses which published vulnerabilities are unpatched |
| Credentials | Passwords, hashes, API keys, SSH private keys, certificates | Obvious, and the most common accidental leak in lab repositories |
| Personal data | Real user names, e-mail addresses, file paths containing a user profile | Identifies the operator |
| Raw diagnostic exports | DxDiag, HWiNFO, CrystalDiskInfo, MSInfo output | Contain most of the above simultaneously |

## 2. Safe to commit

| Category | Examples |
|---|---|
| Component classes | CPU model, RAM capacity and type, drive class and capacity |
| Major OS version | "Windows 11 Pro, 24H2 branch" — without revision |
| Internal lab addressing | RFC1918 ranges used *inside* the virtual lab |
| Design decisions and reasoning | The content this repository exists for |
| Firewall rule intent | The policy table, not the exported configuration file |

Internal lab addressing is publishable because those networks exist only inside the
hypervisor, are not reachable from outside, and are necessary for anyone reproducing
the build. Host LAN addressing is a different matter and is never published.

## 3. Evidence handling

Screenshots and command output are the highest-risk content in a lab repository,
because they leak identifiers the author never intended to include.

Before committing anything to `evidence/`:

1. Crop or redact window titles, taskbars, host names and user profile paths.
2. Replace real host names with role names — `DC01`, `CLIENT01`, `SIEM01`.
3. Remove MAC addresses and any public IP from `ipconfig` / `ip a` output.
4. Strip credentials from configuration exports before saving them.
5. Re-read the file after redacting. Redaction is routinely done once and incompletely.

Packet captures are never committed. They contain everything at once.

## 4. Git history

Deleting a file in a later commit does **not** remove it from the repository. The
content remains retrievable in history and in any clone or fork already taken.

Therefore:
- Sanitise **before** the first commit, not afterwards.
- If a secret is committed, treat it as compromised: rotate or revoke it, then rewrite
  history. Rewriting alone is not sufficient.
- `.gitignore` blocks accidental commits by pattern; it is a safety net, not a policy.
  Patterns cover VM disk images, ISOs, keys, packet captures and raw log exports.

## 5. Review before every commit

```
git status
git diff --staged
```

Reading the staged diff before committing takes seconds and catches the majority of
accidental disclosures. It is the single highest-value habit in this policy.

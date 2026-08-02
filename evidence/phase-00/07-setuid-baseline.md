# Phase 0 — Evidence 07: setuid baseline

**Purpose:** record the set of setuid binaries present on `LAB01` in a known-good
state, so that later additions can be detected by comparison.
**Date:** 2026-08-02

---

## Why

A setuid binary runs with the privileges of its owner rather than of the user who
launches it. Enumerating them is a standard early step after obtaining limited
access to a machine, because any flaw in one is a path to higher privilege.

The defensive counterpart is not to eliminate them — several are required for
ordinary operation — but to know which ones are supposed to be there. **A setuid
binary appearing where none was before is among the strongest indicators of
compromise.**

## Command

```bash
find / -perm -4000 -type f 2>/dev/null | sort > ~/setuid-baseline.txt
wc -l ~/setuid-baseline.txt
```

`-perm -4000` matches the setuid bit. `-type f` restricts the search to regular
files. `2>/dev/null` discards the permission errors produced by searching
directories the user cannot read.

Result on a clean Ubuntu Server 26.04 installation: **15 files.**

## Binaries found under /usr/bin

| Binary | Why it is setuid |
|---|---|
| `passwd`, `chsh`, `chfn`, `gpasswd` | Write to root-owned account files |
| `su`, `sudo.ws`, `newgrp` | Change identity — privilege is the function |
| `mount`, `umount`, `fusermount3`, `ntfs-3g` | Mounting filesystems is a kernel operation |

Each is defensible on the same test: **why does this program need root?** A binary
that cannot answer that question is worth investigating.

## Comparison procedure

```bash
find / -perm -4000 -type f 2>/dev/null | sort > /tmp/setuid-now.txt
diff ~/setuid-baseline.txt /tmp/setuid-now.txt
```

`diff` produces no output when the files match. In Unix, silence is the
confirmation — the same convention as `sshd -t`, which reports nothing when a
configuration is valid.

## Scope and limits

This baseline covers setuid binaries only. It does not cover setgid, file capabilities,
sudoers configuration, or any other privilege mechanism. It is a single narrow check,
recorded because it is cheap and because the comparison method generalises.

The method itself is the transferable part: **record the normal state, compare
against it later, investigate the difference.** Applied to one file it is `diff`.
Applied to a whole estate it is what a SIEM does.

## Storage

The baseline is kept in version control rather than on the lab machine. A reference
copy that lives only on the system it describes is lost with that system — and, on a
machine with no recycle bin and no file versioning, lost to a single mistyped
command.

# Phase 0 — Evidence 02: Installation media checksum

**Exit criterion 3, part 1 of 2:** installation media must be checksum-verified before
use.
**Date:** 2026-07-29

---

## Media

| Field | Value |
|---|---|
| Image | Ubuntu Server 26.04 LTS, `live-server-amd64` |
| Stored as | `UbuntuServer26.04.iso` (renamed; the hash covers content, not file name) |
| Expected SHA-256 | `dec49008a71f6098d0bcfc822021f4d042d5f2db279e4d75bdd981304f1ca5d9` |

## Command (Git Bash)

```bash
echo "dec49008a71f6098d0bcfc822021f4d042d5f2db279e4d75bdd981304f1ca5d9 *UbuntuServer26.04.iso" | sha256sum --check
```

## Output

```
UbuntuServer26.04.iso: OK
```

## Limits of this verification

SHA-256 establishes **integrity**: the file was not corrupted or altered in transit.

It does **not** establish **authenticity**. Ubuntu is distributed through dozens of
third-party mirrors. If a mirror were compromised, an attacker could publish both a
modified image and a matching hash.

The control that closes that gap is GPG verification of the `SHA256SUMS` file against
Canonical's signing key:

```bash
gpg --verify SHA256SUMS.gpg SHA256SUMS
```

**This was not performed.** Recorded as a known gap rather than left implicit.

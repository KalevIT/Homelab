# Phase 0 — Evidence 05: SSH key-based authentication

**Exit criterion 5:** SSH key-based authentication working; password authentication
disabled.
**Date:** 2026-07-30, corrected 2026-08-02

> **This document was wrong when first written.** The criterion was recorded as met on
> 2026-07-30 while password authentication was in fact still enabled. See
> *Correction* at the end — the reasoning error is the useful part.

---

## Why this control

Password authentication can be attacked by guessing: an attacker with network access
can try candidates indefinitely, and the only limits are rate controls and password
strength. Public-key authentication removes that attack surface entirely — there is no
secret to guess, because the private key never crosses the network.

This is the first hardening control applied in this lab. The reasoning appears directly
in the CompTIA Security+ syllabus under authentication methods.

## Key generated (Windows host)

```
ssh-keygen -t ed25519 -C "lab-access"
```

| Field | Value |
|---|---|
| Algorithm | Ed25519 |
| Fingerprint | `SHA256:WInRCvFxl6jE96+xuJN9/pxRwbUbOpgP1QvA62/NPeU` |
| Private key | `%USERPROFILE%\.ssh\id_ed25519` — never moved, never copied |
| Public key | `%USERPROFILE%\.ssh\id_ed25519.pub` |

Ed25519 was chosen over RSA: shorter keys, faster verification, and no parameter
choices to get wrong.

## Public key installed on LAB01

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys      # public key pasted as a single line
chmod 600 ~/.ssh/authorized_keys
```

Permissions matter: `sshd` refuses to use `authorized_keys` if the file or its
directory is writable by anyone other than the owner. A silent failure here presents
as "the key just doesn't work".

## Password authentication disabled

In `/etc/ssh/sshd_config`:

```
PasswordAuthentication no
PubkeyAuthentication yes
```

Configuration validated **before** applying it:

```bash
sudo sshd -t
```

No output, therefore no syntax error. Only then:

```bash
sudo systemctl restart ssh
```

**Order of operations.** The existing SSH session was kept open throughout, and the new
configuration was tested from a second window. A syntax error in `sshd_config` combined
with a closed session would have left the machine reachable only from the VMware
console. Testing a lockout mechanism while holding the exit open is the same principle
as the snapshot test: prove the new path before removing the old one.

## Verification

Two separate claims have to be proven, and proving one does not prove the other:

1. key-based authentication works
2. password authentication is refused

**Test 1 — the key works.** No prompt appears; the session opens directly:

```
ssh user@<lab-address>
```

```
Welcome to Ubuntu 26.04 LTS (GNU/Linux 7.0.0-28-generic x86_64)
user@lab01:~$
```

**Test 2 — the password is refused.** Public-key authentication is disabled for this
attempt, forcing the server to fall back:

```
ssh -o PubkeyAuthentication=no user@<lab-address>
```

```
user@<lab-address>: Permission denied (publickey).
```

The server refuses rather than prompting. If a password prompt appears here, password
authentication is still enabled regardless of what the configuration file says.

**Configuration confirmed as loaded, not as written:**

```bash
sudo sshd -T | grep -i passwordauthentication
```

```
passwordauthentication no
```

**Criterion met.**

---

## Recorded decisions

### Passphrase intentionally left empty

The private key is not encrypted at rest. Anyone with access to the Windows user
profile can use it without further authentication.

Accepted for this machine: LAB01 is a laboratory host on an internal network, holding
nothing of value. Recorded here because an accepted risk and an overlooked risk are
indistinguishable unless one is written down.

**Standing rule for this lab:** any key that reaches something outside the lab —
version control, a real server, a cloud account — carries a passphrase, with
`ssh-agent` handling the repetition. No exceptions.

### Correction: the criterion was recorded as met before it was

On 2026-07-30 this document recorded the criterion as satisfied. It was not. Password
authentication remained enabled for three days.

**What was verified:** that key-based login succeeded without a prompt.

**What was concluded:** that password authentication was disabled.

The conclusion did not follow. SSH offers authentication methods in order and stops at
the first that succeeds. The key was offered first and worked, so no prompt appeared —
which says nothing about whether a password would also have been accepted.

**How it surfaced.** During a later exercise the home directory was temporarily set to
mode 777. `sshd` then refused the key — correctly, because a world-writable home means
`authorized_keys` can be replaced by anyone — and **fell back to a password prompt,
which succeeded.** The fallback should not have existed.

**Root cause.** Ubuntu's `sshd_config` carries an `Include` directive for
`/etc/ssh/sshd_config.d/*.conf` at line 24. The manual edit was made at line 78.
Cloud-init had written `PasswordAuthentication yes` into
`50-cloud-init.conf` during installation, and for most SSH options **the first value
read wins**. The included file was read first; the later edit was ignored.

No error was produced. The file said `no`, the service used `yes`.

**Fix.** The directive in `50-cloud-init.conf` was corrected, validated with
`sshd -t`, confirmed with `sshd -T`, and only then applied with a service restart. The
existing session was held open throughout.

**Lessons recorded rather than tidied away:**

1. **`sshd -T` reports the loaded configuration; the file reports an intention.** When
   a setting matters, the loaded value is the one to check.
2. **A control is not verified until its failure path is tested.** "The key works" and
   "the password is refused" are separate claims requiring separate tests.
3. **A configuration believed applied but silently overridden is worse than one never
   attempted**, because it produces confidence without protection. Nothing signals the
   discrepancy until something exercises it.
4. Drop-in configuration directories are ordinary on modern distributions. Editing the
   main file is not sufficient on its own.

### Gap found in `.gitignore`

The credentials section matched `id_rsa*` only, a pattern predating Ed25519. The key
generated here is named `id_ed25519` and would not have been excluded — a copy landing
in the working tree by accident would have been committed silently, to a repository
that is now public.

Patterns extended to cover `id_ed25519*`, `id_ecdsa*`, `*.pub`, `known_hosts*` and
`authorized_keys`.

The wider lesson: a `.gitignore` written once and never revisited protects against
yesterday's mistakes. It is reviewed whenever a new class of sensitive file enters the
working environment.

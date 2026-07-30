# Phase 0 — Evidence 05: SSH key-based authentication

**Exit criterion 5:** SSH key-based authentication working; password authentication
disabled.
**Date:** 2026-07-30

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

Login before the change — password prompted:

```
user@192.168.231.129's password:
```

Login after the change — no prompt, authenticated by key:

```
Welcome to Ubuntu 26.04 LTS (GNU/Linux 7.0.0-28-generic x86_64)
Last login: Thu Jul 30 20:44:33 2026 from 192.168.231.1
user@lab01:~$
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

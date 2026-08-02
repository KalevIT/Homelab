# Permissions, Baselines and Network Services — Consolidation Exercises

Three sessions of roughly one hour each, worked through on the lab machine.

Companion to [`cli-fundamentals.md`](cli-fundamentals.md), which covers the earlier
material. Same structure and the same rules: one concept per session, exercises that
verify themselves, failures induced on purpose, and verification questions that are
posed but not answered.

---

## Why this document exists

The second half of the wargame introduced a different class of problem. The earlier
levels named their target: *that* file, *that* port. The later ones did not — they
required building the information first and acting on it second.

Three subjects account for most of it: how privilege is granted on Unix, how change
is detected by comparison, and how network services are discovered and classified.
All three recur throughout the remaining phases of this lab, so they are worth
holding properly rather than approximately.

## Setup

```bash
mkdir ~/exercises2
cd ~/exercises2
```

---

# Session F — Permissions and privilege

## Why it matters

Every access decision on Unix reduces to permissions. Reading them correctly is not
optional, and reading them *incorrectly* is worse than not reading them at all —
it produces confident wrong conclusions.

This session also covers setuid, which is the mechanism behind most local privilege
escalation.

## Part 1 — Reading permissions

**F1.** Create a file and look at it:

```bash
echo "content" > sample.txt
ls -l sample.txt
```

The first field is ten characters. Read it in four parts:

```
-  rw-  rw-  r--
│   │    │    └── others
│   │    └─────── group
│   └──────────── owner
└──────────────── file type
```

**F2.** Check the type character on different objects:

```bash
ls -ld /etc          # d — directory
ls -l  /etc/passwd   # - — regular file
ls -l  /dev/null     # c — character device
ls -l  /bin/ls       # l — symbolic link, if present
```

That leading character is the one most often skimmed past. `/dev/null` is not a file
that discards writes; it is a device that does.

## Part 2 — Changing permissions

**F3.** Symbolic notation — add and remove one bit at a time:

```bash
chmod u+x sample.txt
ls -l sample.txt
chmod u-x sample.txt
ls -l sample.txt
```

`u` owner, `g` group, `o` others, `a` all. `+` adds, `-` removes, `=` sets exactly.

**F4.** Numeric notation — set all three groups at once:

```bash
chmod 644 sample.txt
ls -l sample.txt
chmod 600 sample.txt
ls -l sample.txt
```

Each digit is the sum of read 4, write 2, execute 1:

| Digit | Meaning |
|---|---|
| 7 | rwx |
| 6 | rw- |
| 5 | r-x |
| 4 | r-- |
| 0 | --- |

`600` is the mode SSH requires on a private key: owner reads and writes, nobody else
sees it at all.

**F5.** Observe the rule being enforced. Loosen a key's permissions and watch SSH
refuse it:

```bash
chmod 644 ~/.ssh/id_ed25519
ssh -o BatchMode=yes localhost true
chmod 600 ~/.ssh/id_ed25519
```

The refusal is deliberate: a private key readable by others is treated as
compromised, so the tool declines rather than proceeding.

## Part 3 — Directories behave differently

**F6.** On a directory, `x` does not mean "execute":

```bash
mkdir testdir
echo "inside" > testdir/file.txt
chmod 644 testdir          # r and w, no x
ls testdir
cat testdir/file.txt
```

`ls` works — `r` allows listing the names. `cat` fails — without `x` the directory
cannot be traversed to reach the contents.

**F7.** Now the reverse:

```bash
chmod 711 testdir
ls testdir
cat testdir/file.txt
```

`ls` fails, `cat` succeeds. **On a directory, `r` lists names and `x` grants passage.**
They are independent, and a directory that grants `x` without `r` holds files that can
be read by anyone who already knows their names.

**F8.** Restore and continue:

```bash
chmod 755 testdir
```

## Part 4 — setuid

**F9.** Look at the canonical example:

```bash
ls -l /usr/bin/passwd
```

The `s` sits where the owner's `x` would be. It means: **this program runs with the
privileges of its owner, not of whoever launched it.**

Changing your own password requires writing to a root-owned file. setuid is what
makes that possible without granting administrative rights.

**F10.** Enumerate every setuid binary on the system:

```bash
find / -perm -4000 -type f 2>/dev/null | sort
```

**F11.** For each one, ask the same question: *why does this program need root?*

Account tools write to root-owned files. Identity tools change identity, which is the
privilege itself. Mount tools perform kernel operations. Each has an answer. **A
binary that has no answer is worth investigating.**

**F12.** Record the baseline and verify the comparison works:

```bash
find / -perm -4000 -type f 2>/dev/null | sort > baseline.txt
find / -perm -4000 -type f 2>/dev/null | sort > now.txt
diff baseline.txt now.txt
```

No output means no difference. In Unix, silence is the confirmation.

**F13.** Simulate an addition and confirm the check catches it:

```bash
echo "/usr/bin/fake-setuid-binary" >> now.txt
diff baseline.txt now.txt
```

This is the entire principle of change detection, at the smallest possible scale.

## Part 5 — The other two special bits

**F14.** setgid — same idea, applied to the group:

```bash
find / -perm -2000 -type f 2>/dev/null | head
```

**F15.** The sticky bit — a different mechanism entirely:

```bash
ls -ld /tmp
```

The trailing `t` means: anyone may create files here, but only the owner of a file
may delete it. Without it, a world-writable directory would let any user remove any
other user's files.

## Verification

1. Read `-rwsr-x---` aloud: what may the owner do, the group, others?
2. What is the numeric equivalent of `rw-------`, and where is that mode required?
3. On a directory, what do `r` and `x` each permit?
4. What does setuid do, and how does it differ from `sudo`?
5. Why is a baseline of setuid binaries worth keeping?
6. Give a legitimate reason for a setuid binary to exist, and a reason it is a risk.

---

# Session G — Baselines and change detection

## Why it matters

Most detection reduces to one question: **is this different from how it should be?**

Answering it requires two things — a record of how it should be, and a way to compare.
Neither is difficult. What is difficult is having recorded the baseline *before*
needing it.

## Part 1 — diff

**G1.** Create two nearly identical files:

```bash
printf 'alpha\nbravo\ncharlie\ndelta\n' > original.txt
printf 'alpha\nbravo\ncharlie\ndelta\necho compromised\n' > modified.txt
diff original.txt modified.txt
```

One appended line, found immediately. This is the shape of the shell-startup-file
problem: a long ordinary file with something added at the end.

**G2.** Change a line in the middle instead:

```bash
printf 'alpha\nbravo\nCHANGED\ndelta\n' > altered.txt
diff original.txt altered.txt
```

Read the notation: `<` is the first file, `>` is the second, and the numbers give the
line positions.

**G3.** A side-by-side view is easier for configuration files:

```bash
diff -y original.txt altered.txt
```

**G4.** Sometimes only the answer matters, not the detail:

```bash
diff -q original.txt altered.txt
diff -q original.txt original.txt
```

**G5.** Invisible differences are real differences:

```bash
printf 'alpha\n\tbravo\n' > tabbed.txt
printf 'alpha\n    bravo\n' > spaced.txt
diff tabbed.txt spaced.txt
```

The two files look identical on screen. `diff` compares bytes, not appearance.

**G6.** When whitespace is noise rather than signal, exclude it:

```bash
diff -w tabbed.txt spaced.txt
```

Both behaviours are correct. Which one is wanted depends on the question.

## Part 2 — Comparing directories

**G7.** `diff` works on directories as well as files:

```bash
mkdir dir1 dir2
echo "same" > dir1/a.txt
echo "same" > dir2/a.txt
echo "only here" > dir1/b.txt
diff -r dir1 dir2
```

It reports both differing contents and files present on one side only.

## Part 3 — Comparing by fingerprint

**G8.** For large files or long lists, comparing a hash is faster than comparing
content:

```bash
sha256sum original.txt altered.txt
```

Two different values mean two different files. Identical values mean identical
content, whatever the name or timestamp.

**G9.** Record and verify a set of files in one operation:

```bash
sha256sum *.txt > checksums.txt
sha256sum -c checksums.txt
```

**G10.** Now break one and re-check:

```bash
echo "tampered" >> original.txt
sha256sum -c checksums.txt
```

The failure names the file. This is the same control applied to installation media
in Phase 0, used here for change detection instead.

**Note the limit, which is the same one recorded for the ISO:** a hash establishes
that content changed, not who changed it or whether the recorded hash is itself
trustworthy. A baseline stored where an attacker can edit it proves nothing.

## Part 4 — Where change matters

**G11.** The files that execute automatically are the ones worth watching:

```bash
ls -la ~ | grep -E '^\.|bash|profile'
```

**G12.** Check the end of a startup file, which is where appended lines land:

```bash
tail -5 ~/.bashrc
```

**G13.** Compare against the distribution default:

```bash
diff ~/.bashrc /etc/skel/.bashrc
```

`/etc/skel` holds the templates copied into every new user's home directory. It is a
baseline that exists without anyone having created it.

Differences here are expected — personal customisation lives in this file. The point
is knowing which differences are yours.

## Verification

1. In `diff` output, what do `<` and `>` mean?
2. Why can two visually identical lines be reported as different?
3. When would `-w` be correct, and when would it hide something important?
4. What does a matching hash prove, and what does it not prove?
5. Why is a baseline stored on the machine it describes a weak control?
6. Which files on a Linux system execute automatically, and why does that matter?

---

# Session H — Network service discovery

## Why it matters

Before anything can be secured, it has to be known to exist. The first question about
any machine is what it exposes, and the second is whether what answers is what it
claims to be.

This session is directly preparatory: verifying that a firewall rule blocks what it
claims to block is the same procedure applied to a different target.

## Part 1 — Local view

**H1.** What is listening on this machine:

```bash
sudo ss -tulpn
```

Without `sudo` the owning process is hidden. The same command returns different
information depending on who runs it.

**H2.** For each listening port, ask what it is and whether it should be there. On a
minimal server install the list is short, and every entry should have a reason.

**H3.** Established connections rather than listeners:

```bash
ss -tp
```

`-l` shows what is waiting for connections; without it, what is currently connected.

## Part 2 — Remote view

**H4.** Test a single port:

```bash
nc -zv localhost 22
nc -zv localhost 9999
```

`succeeded` and `refused` are different answers, and both are informative. A third
outcome — a hang with no answer at all — usually means a firewall is dropping the
traffic silently rather than rejecting it.

**H5.** Scan a range:

```bash
nmap -p 20-25 localhost
```

**H6.** Scan the whole standard range and time it:

```bash
time nmap localhost
```

**H7.** Ask what the services actually are:

```bash
nmap -sV -p 22 localhost
```

Compare the duration with the previous scan. A plain scan asks "is this port open?".
Version detection opens the connection and makes the service speak, which takes
considerably longer.

**H8.** Read the VERSION column critically. It is derived from how the service
responded to probes — a hypothesis, not a fact. A service can be configured to
report anything. **A tool indicates where to look; confirming is a separate step.**

### Scope

Scanning is legitimate against systems one owns or is authorised to test. Against
anything else it is unlawful in most jurisdictions, including Italy. In this lab the
targets are the lab's own virtual machines. That boundary does not move.

## Part 3 — Confirming by hand

**H9.** Connect to a plaintext service and observe it directly:

```bash
nc -zv localhost 22
```

**H10.** Connect to a TLS service and read the negotiation:

```bash
echo | openssl s_client -connect ubuntu.com:443 2>/dev/null | grep -iE "protocol|cipher|verify"
```

**H11.** Contrast a valid chain with a self-signed certificate. Any service using a
self-signed certificate — a router's management interface, a lab appliance — produces
`Verify return code: 18`.

The connection is still encrypted. What is missing is any assurance about who is
answering.

## Part 4 — Recording the state

**H12.** Capture the listening state as a baseline:

```bash
sudo ss -tulpn | sort > ~/listening-baseline.txt
```

**H13.** The comparison is the same one from Session G:

```bash
sudo ss -tulpn | sort > /tmp/listening-now.txt
diff ~/listening-baseline.txt /tmp/listening-now.txt
```

A port that appears without explanation is worth an answer. This becomes materially
more useful once the lab holds several machines.

## Verification

1. What is the difference between `ss` and `nmap`, and when is each appropriate?
2. Why does `ss -tulpn` show different output under `sudo`?
3. What do `succeeded`, `refused` and a silent hang each indicate?
4. Why is a version-detection scan slower than a plain port scan?
5. Why should a service identification be confirmed by hand?
6. Under what circumstances is scanning a system lawful?

---

# Closing

```bash
cd ~
rm -r ~/exercises2
```

Reread every verification question from all three sessions in sequence. Those
answered without hesitation are settled; the others indicate where to return.

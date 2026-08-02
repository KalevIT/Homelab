# Phase 0 — Evidence 06: OverTheWire Bandit

**Exit criterion 6:** OverTheWire *Bandit* levels 0-20 completed.
**Period:** 2026-07-30 to 2026-08-02
**Status:** met

---

## Scope and disclosure

OverTheWire asks that solutions and passwords not be published, and states this
explicitly in the server login banner. This document therefore records **what each
level taught**, not how it was solved.

Naming a tool is not a disclosure — the level pages list the suggested tools
themselves. Publishing the arguments that solve a level is. That line is held
throughout.

---

## What the levels covered

| Levels | Subject |
|---|---|
| 0-3 | Shell basics: hidden files, awkward filenames, argument parsing |
| 4-6 | Identifying files by type and attribute rather than by name |
| 7-9 | Filtering and counting text; extracting strings from binaries |
| 10-11 | Encoding versus encryption |
| 12 | Layered compression and archive formats |
| 13, 16 | Public-key authentication from the client side |
| 14-16 | Network services, plaintext and TLS, port discovery |
| 17-18 | File comparison; shell startup files as a persistence mechanism |
| 19 | setuid binaries and privilege escalation |

---

## Concepts worth recording

### A file's name says nothing about its contents

Established at level 4 and reinforced at level 12, where a file compressed through
eight layers never once announced its format correctly. `file` reads the leading
bytes and answered correctly every time; the name and extension never did.

The corollary matters more than the tool: **an attachment named `invoice.pdf` can be
an executable.** Establishing what a file actually is precedes opening it.

### Attributes and content are different problems

`find` searches *for* files by their attributes — owner, group, size, permissions,
timestamps. `grep` searches *inside* files for content.

Choosing the wrong one is the most common reason a search returns nothing. The
question to ask first is whether the criterion is a property of the file or of what
it contains.

### Output travels on two channels

Results and errors are separate streams and can be redirected independently. Sending
errors away leaves only real results on screen.

The failure mode is worth naming: **a command that produced nothing and a command
whose results were discarded look identical.** The exit code distinguishes them.

### Encoding is not encryption

Base64 and ROT13 have no key. Anyone reverses them. They exist to move data through
channels that only accept text, or to make it non-obvious, not to protect it.

Three categories that must stay distinct:

| | Purpose | Key | Reversible |
|---|---|---|---|
| Encoding | Transport | none | by anyone |
| Encryption | Confidentiality | yes | with the key only |
| Hashing | Integrity | none | not at all |

This matters operationally: base64 is routinely used to obscure payloads in logs.
Treating it as encryption means assuming a key is needed, and stopping.

### TLS provides encryption and identity, and they are separable

Connecting to a service by hand shows what a browser hides. The server presents a
certificate; the client checks whether the chain terminates in something already
trusted.

One level's server used a self-signed certificate. The connection was fully
encrypted. It established nothing about *who* was answering — an interceptor can
present their own self-signed certificate and the client cannot tell.

That is what a browser warning actually reports: not "this is unencrypted" but "I
cannot confirm who this is".

### Automated identification is a hypothesis

A port scan with service detection labelled one port differently from the rest. That
label came from how the service responded to probes, not from certainty.
Confirming it by hand is the step that turns it into a fact.

**A tool indicates where to look. It does not decide what to conclude.**

Also recorded: the useful information appeared inside output that looked like
technical noise — a fingerprint block the scanner prints when it *fails* to identify
a service. Reading it before deciding to filter it was the correct order.

### Startup files are a persistence mechanism

One level modified a shell startup file so that any interactive login was
disconnected immediately. The file was otherwise 120 lines of ordinary distribution
default; two lines had been appended at the end.

That file executes automatically on every interactive login. Anything written into
it restarts itself without further effort by whoever put it there.

Detecting it is a comparison problem: the modified file is indistinguishable from
normal until placed next to a known-good copy. The preceding level taught exactly
that comparison, which is not a coincidence.

### setuid is privilege escalation by design

A `s` in place of the owner's execute bit means the program runs with **the owner's**
privileges regardless of who launches it.

This is not `sudo`. `sudo` requires explicit authorisation and produces an audit
record. setuid is a file attribute: no prompt, no log, granted automatically to
anyone holding execute permission.

It exists for good reasons — changing one's own password requires writing to a
root-owned file — and it is a standing risk, because it is code running with
elevated privilege on behalf of an unprivileged caller. A setuid binary that accepts
an arbitrary command hands its owner's privileges to whoever runs it.

Enumerating setuid binaries is among the first actions taken after obtaining limited
access to a machine. The defensive counterpart is a recorded baseline, compared
periodically — see `07-setuid-baseline.md`.

---

## Method notes

**What worked.** Establishing what a thing is before attempting to act on it.
Three commands — identify the type, check the size, count the lines — cost ten
seconds and change the approach.

**What did not.** Changing options at random when a command failed. The fault was
almost always structural rather than an incorrect flag, and the error message
usually named it. On one level the message stated plainly which file was being read;
several minutes were spent trying flags before it was read properly.

**Where shortcuts were taken.** Two early levels were solved by inspecting files one
at a time rather than using the intended tool. This works at a scale of ten files.
It will not work in Phase 4, where log volume is measured in tens of thousands of
lines per day. Recorded here rather than tidied away.

**A pause was taken at level 15.** Seven tools had been used across seven consecutive
levels, each introduced by a puzzle requiring immediate solution. Every level was
completed, which is not the same as having learned seven tools. Practice exercises
were written and worked through on the lab machine before continuing — see
`docs/exercises/`.

---

## Verification

Progress reached level 20, satisfying the criterion. Level 20 onward is outside the
scope defined in `ROADMAP.md` and was not attempted.

**Criterion met.**

# Phase 0 — Evidence 08: shell startup file baseline

**Purpose:** record the content of the files that execute automatically on login,
while the system is known clean, so that later modification can be detected.
**Date:** 2026-08-03

---

## Why

A file that runs on every login restarts whatever is written into it. Nobody has to
do anything further — the system does the work. This makes startup files one of the
cheapest available persistence mechanisms, and one of the first places to look on a
machine under suspicion.

Detection is a comparison problem. A modified startup file is indistinguishable from
a normal one until it is placed beside a known-good copy: the file is otherwise
ordinary distribution default, and the addition sits at the end where nothing draws
attention to it.

## Timing

This baseline was taken while `~/.bashrc` was still byte-identical to
`/etc/skel/.bashrc`, the distribution template. That will not remain true — aliases
and personal configuration accumulate — and once it stops being true, separating
one's own changes from someone else's becomes harder.

**The value of a baseline is set by when it was taken, not by how it was taken.**

## Command

```bash
sha256sum ~/.bashrc ~/.profile ~/.bash_logout > ~/startup-baseline.txt
ls -la ~ | grep -E '^\.|bash|profile' >> ~/startup-baseline.txt
```

Two records, because they answer different questions: the hashes detect *changed
content*, the listing detects *added or removed files*.

Note the `>>` on the second line. Using `>` overwrites rather than appends, which
silently discarded the hashes on the first attempt — the file looked complete and
contained only half of what was intended.

## Verification

```bash
sha256sum -c ~/startup-baseline.txt
```

Reports `OK` per file, or names the file that no longer matches.

## What was found in the files themselves

Reading the files before recording them turned up three things worth keeping.

### `.profile` executes `.bashrc`

```sh
if [ -f "$HOME/.bashrc" ]; then
    . "$HOME/.bashrc"
fi
```

They are not independent. A login runs `.profile`, which sources `.bashrc`, so a
modification in one file is reached through both paths.

### `.bashrc` sources a file that does not exist

```sh
if [ -f ~/.bash_aliases ]; then
    . ~/.bash_aliases
fi
```

`~/.bash_aliases` is absent on this system. If it were created, it would execute on
every interactive session — **and `.bashrc` itself would still match this baseline.**

A content baseline cannot detect this. Only the file listing can, which is why both
are recorded. It is a concrete demonstration that a control has to be matched to the
thing it is meant to catch.

### `.bashrc` exits early when non-interactive

```sh
case $- in
    *i*) ;;
      *) return;;
esac
```

This explains a wargame level solved earlier without understanding why. That level's
`.bashrc` ended with a logout command, yet running a single command over SSH still
worked — because a non-interactive invocation returns at this check, before reaching
the end of the file.

## Scope and limits

Covers three per-user files only. **Not covered:**

- system-wide equivalents in `/etc/profile`, `/etc/bash.bashrc`, `/etc/profile.d/`
- scheduled execution via `cron` or systemd timers
- systemd unit files
- any other account on the machine

Recorded as a known gap rather than left implicit. The method generalises; this
application of it is deliberately narrow.

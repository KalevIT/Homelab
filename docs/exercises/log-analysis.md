# Log Analysis with journalctl — Consolidation Exercises

One session of roughly an hour, worked through on the lab machine.

Third in the series, after [`cli-fundamentals.md`](cli-fundamentals.md) and
[`permissions-and-services.md`](permissions-and-services.md). Same rules: one concept
per session, exercises that verify themselves, failures induced on purpose, and
verification questions posed but not answered.

---

## Why this document exists

The Phase 0 exit test asks a question about authentication failures in the system
log. It was attempted before this session existed, and the attempt exposed a gap in
the sequencing rather than in the material: `journalctl` appears in the Phase 0
curriculum but had never been practised. Every other tool in this phase was exercised
before being required.

This session closes that gap. It also covers two errors found during the attempt,
both of which produce plausible wrong answers rather than visible failures — which is
what makes them worth practising deliberately.

## Setup

```bash
mkdir ~/exercises3
cd ~/exercises3
```

---

## Part 1 — What the journal is

**I1.** Look at the whole thing, then leave:

```bash
journalctl
```

It opens in a pager. `q` exits, arrow keys and `Page Down` scroll, `G` jumps to the
end.

**I2.** The journal is not a text file. Confirm it:

```bash
ls -la /var/log/journal/*/
file /var/log/journal/*/system.journal
```

Entries are stored in a structured binary format, not as lines of text. `cat` on
those files produces nothing useful; `journalctl` is the interface to them.

**I3.** This matters because entries carry **fields**, not just text:

```bash
journalctl -n 1 -o verbose
```

One entry, fully expanded. Note `_PID`, `_UID`, `_SYSTEMD_UNIT`, `_HOSTNAME`,
`MESSAGE` and the rest. A traditional log line is a string; a journal entry is a
record that happens to have a message in it.

**I4.** Compare the compact form:

```bash
journalctl -n 1
```

The default output shows timestamp, host, process, PID and message. Everything else
is still there, just not displayed.

---

## Part 2 — Filtering by time

**I5.** Relative expressions:

```bash
journalctl --since "1 hour ago" | wc -l
journalctl --since "24 hours ago" | wc -l
journalctl --since yesterday | wc -l
```

**I6.** Absolute, and bounded at both ends:

```bash
journalctl --since "2026-08-03 20:00" --until "2026-08-03 21:00" | wc -l
```

**I7.** `-S` and `-U` are the short forms of `--since` and `--until`. Confirm they
agree:

```bash
journalctl -S "1 hour ago" | wc -l
journalctl --since "1 hour ago" | wc -l
```

**I8.** Check which timezone the output uses:

```bash
timedatectl
journalctl -n 1
```

A server set to UTC displays UTC. When a report says "in the last 24 hours", it is
worth knowing whose clock that is — the discrepancy is invisible until two sources
are compared.

---

## Part 3 — Filtering by source and severity

**I9.** By unit, which is more precise than searching the text:

```bash
journalctl -u ssh --since "24 hours ago" | wc -l
journalctl -u ssh -n 20
```

**I10.** By priority. The scale runs 0 (emergency) to 7 (debug), and a given level
includes everything more severe:

```bash
journalctl -p err --since "24 hours ago"
journalctl -p warning --since "24 hours ago" | wc -l
```

**I11.** Filters combine, and each one narrows further:

```bash
journalctl -u ssh -p warning --since "24 hours ago"
```

**I12.** Compare a native filter against a text search:

```bash
journalctl -u ssh --since "1 hour ago" | wc -l
journalctl --since "1 hour ago" | grep -c "sshd" | cat
```

They are not equivalent. `-u ssh` selects entries the unit actually produced;
`grep "sshd"` selects entries whose *text* contains that string, which can include
messages emitted by something else that merely mentions it.

**Prefer a native filter over a text match when one exists.** A text match is a
guess about how the message is worded.

**I13.** Follow the journal live. Open a second session, log in over SSH, and watch:

```bash
journalctl -u ssh -f
```

`Ctrl+C` stops it. This is the shape of monitoring: not reading a file, but watching
events arrive.

---

## Part 4 — From lines to events

This part covers the first of the two errors mentioned above.

**I14.** Generate some failures. From the Windows host, attempt a login with a
username that does not exist, several times:

```
ssh nosuchuser@<lab-address>
```

**I15.** Also generate a local authentication failure, which produces a different
message shape:

```bash
sudo -k
sudo whoami
```

Enter a wrong password when prompted.

**I16.** Now look at what SSH recorded:

```bash
journalctl -u ssh --since "1 hour ago" | grep -i "invalid user"
```

**I17.** Read the PID in brackets on each line. **Each PID appears twice.**

One failed attempt produces two entries: one stating the user does not exist, one
recording that the connection closed. Both contain the phrase searched for.

**I18.** Count both ways and compare:

```bash
journalctl -u ssh --since "1 hour ago" | grep -ci "invalid user"
journalctl -u ssh --since "1 hour ago" | grep -c "Invalid user"
```

The first is case-insensitive and matches both entries. The second matches only the
first, because the closing message writes the word in lower case mid-sentence.

**Counting log lines is not counting events.** This is the most common error in log
analysis and it produces plausible numbers, which is precisely why it survives
review. A report stating eleven failed attempts when there were six is wrong, and
nobody notices, because eleven is a believable figure.

**I19.** `-i` is not free. It makes a filter more permissive, which is sometimes
required and sometimes catches what was not wanted. The same option was necessary in
an earlier session to match `Verify` and is harmful here.

**No option is correct in general.** It depends on what is being separated.

---

## Part 5 — Extracting fields

**I20.** Treat a line as whitespace-separated fields and take one:

```bash
journalctl -u ssh --since "1 hour ago" | grep "Invalid user" | cut -d' ' -f10
```

`cut` defaults to tab as its delimiter, so `-d' '` is required. `-f10` selects the
tenth field.

**I21.** Verify the field number rather than assuming it. Print the line and count:

```bash
journalctl -u ssh --since "1 hour ago" | grep "Invalid user" | head -1
```

**I22.** `cut` is positional and breaks when the position shifts. `awk` selects by
field number too, but tolerates runs of whitespace:

```bash
journalctl -u ssh --since "1 hour ago" | grep "Invalid user" | awk '{print $10}'
```

**I23.** `awk` can also select by content rather than position, which does not break
when the message format changes:

```bash
journalctl -u ssh --since "1 hour ago" | awk '/Invalid user/ {print $(NF-2)}'
```

`NF` is the number of fields on the line; `$(NF-2)` counts back from the end. Where
the address is always third from last, this survives changes earlier in the line.

---

## Part 6 — Ordering a pipeline correctly

This part covers the second error, and it is the one that produced a correct answer
for the wrong reason.

**I24.** Establish the behaviour on a small case:

```bash
printf '10.0.0.1\n10.0.0.2\n10.0.0.1\n' | uniq -c
printf '10.0.0.1\n10.0.0.2\n10.0.0.1\n' | sort | uniq -c
```

Without `sort`, the repeated value is counted twice as two separate runs.

**I25.** Now the error, in its realistic form. Sorting the full lines sorts by
**timestamp**, because that is what each line starts with:

```bash
journalctl -u ssh --since "24 hours ago" | grep "Invalid user" | sort | cut -d' ' -f10 | uniq -c
```

**I26.** And the correct order — cut first, then sort, then count:

```bash
journalctl -u ssh --since "24 hours ago" | grep "Invalid user" | cut -d' ' -f10 | sort | uniq -c | sort -rn
```

**I27.** On a small sample both may agree. They agree when each address happens to
appear in a contiguous block of time — which is a property of the data, not of the
command. Interleave two sources and the first version miscounts.

**A command that returns the right answer is not the same as a correct command.**
Establishing *why* an answer is right is the step that distinguishes the two, and it
is the only defence against a pipeline that works until the data changes shape.

**I28.** State the order as a rule: **filter, extract, group, count, rank.** Each
stage does one thing and hands its output to the next. `sort` groups; it never
counts. `uniq -c` counts; it never groups.

---

## Part 7 — Declaring scope

**I29.** The pipeline above counts attempts against **non-existent** users only. A
different message is produced when the username exists and the password is wrong:

```bash
journalctl -u ssh --since "24 hours ago" | grep -c "Failed password"
```

On a host with password authentication disabled this returns zero, because such
attempts cannot occur.

**I30.** Authentication failures also occur outside SSH:

```bash
journalctl --since "24 hours ago" | grep "authentication failure"
```

`sudo` and `su` record their own, through a different mechanism entirely.

A result is only interpretable alongside what it excluded. "Six failed login
attempts" invites the reading that there were no others; "six SSH attempts against
non-existent usernames, on a host where password authentication is disabled" is a
finding. **Stating the boundary of an analysis is part of the analysis**, the same
discipline applied earlier to a checksum that proved integrity but not authenticity.

---

## Verification

1. How does a journal entry differ from a line in a text log file?
2. Why is `-u ssh` preferable to searching the output for `sshd`?
3. Why does one failed SSH login produce two entries, and how can they be told apart?
4. What is the difference between counting lines and counting events?
5. Why must `sort` come after `cut` and not before?
6. A pipeline returns the right answer on today's data. What would have to be true of
   the data for it to return a wrong one?
7. What does `cut -d' ' -f10` assume, and when does that assumption fail?
8. Given a count of failed logins, what should be stated alongside the number?

---

## Closing

```bash
cd ~
rm -r ~/exercises3
```

Reread the verification questions in sequence. Those answered without hesitation are
settled; the others indicate where to return.

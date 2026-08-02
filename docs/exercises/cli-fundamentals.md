# Command Line Fundamentals — Consolidation Exercises

Five sessions of roughly one hour each, worked through on the lab machine.

---

## Why this document exists

Phase 0 uses OverTheWire *Bandit* as structured shell practice. Progress through the
early levels was fast, and then it was not: seven new tools appeared across seven
consecutive levels, each one introduced by a puzzle that had to be solved immediately.

Every level was completed. That is not the same as having learned seven tools.

The failure mode is specific and worth naming: hit a wall, ask for an explanation,
apply it, move to the next level, repeat. Each individual step works. Nothing
accumulates. Three weeks later the tools are recognised but not owned.

These exercises were written to break that loop. They cover the same concepts, on this
lab's own machine, where a mistake costs nothing and can be repeated deliberately.

**Design principles:**

- **One concept per session.** Fits the working rhythm of one hour per day.
- **Exercises produce their own verification.** The output shows whether it worked.
- **Errors are induced on purpose.** Several exercises exist to make something fail,
  because a failure that has been observed is understood better than one that has been
  described.
- **Verification questions are not answered here.** They exist to be answered without
  looking. Hesitating on one indicates which section to repeat.

**Note on scope:** Bandit level numbers are deliberately not mapped to exercises.
OverTheWire asks that solutions not be published, and while naming a tool is harmless —
the level pages list the tools themselves — publishing the arguments that solve a level
is not.

---

## Setup

```bash
mkdir ~/exercises
cd ~/exercises
```

Everything is created inside that directory and removed at the end.

---

# Session A — The two output channels

## Why it matters

Every command on Linux writes to **two separate channels**, not one.

This looks like a detail. It is the mechanism that makes it possible to filter ten
thousand lines of log output down to the part that matters. Without it, the only
option is to read the screen and hope.

## Exercises

**A1.** Run a command that produces both a result and an error:

```bash
ls /etc/passwd /etc/nonexistent
```

Two lines appear. On screen they look identical. They are not.

**A2.** Discard the errors:

```bash
ls /etc/passwd /etc/nonexistent 2>/dev/null
```

**A3.** Now the opposite — discard the results:

```bash
ls /etc/passwd /etc/nonexistent 1>/dev/null
```

Only the error survives. This is what happens when the `2` is forgotten.

**A4.** Save results to a file, leave errors on screen:

```bash
ls /etc/passwd /etc/nonexistent > results.txt
```

**A5.** Inspect the file:

```bash
cat results.txt
```

The error is absent: it travelled on the other channel.

**A6.** Separate both channels into two files:

```bash
ls /etc/passwd /etc/nonexistent > ok.txt 2> errors.txt
```

**A7.** Check each:

```bash
cat ok.txt
cat errors.txt
```

## Exit codes

Every command leaves behind a number when it finishes: `0` on success, non-zero on
failure. It is not displayed, but it is there.

**A8.** Read it with `$?`:

```bash
ls /etc/passwd
echo $?
```

**A9.** Now a failing command:

```bash
ls /etc/nonexistent
echo $?
```

**A10.** The case that matters:

```bash
ls /etc/passwd /etc/nonexistent 2>/dev/null
echo $?
```

The screen looked clean; the exit code says something failed.

**This is how "empty" is distinguished from "broken".** A command that produced no
results and a command whose results were discarded look identical on screen.

`$?` must be read **immediately** after the command. Any subsequent command overwrites
it — including `echo` itself.

## Verification

1. What are the two channels and what number does each have?
2. What does `2>/dev/null` do, and what does `>/dev/null` do?
3. If a command produces an empty screen, what different explanations are possible?
4. Why is `/dev/null` described as a black hole?
5. How do you tell "no results" apart from "results discarded"?

---

# Session B — What files actually are

## Why it matters

On Linux **a file name does not describe its contents**. The extension is a convention
for humans, not information for the system.

Anyone distributing malware knows this: a file called `invoice.pdf` can be an
executable. The first thing an analyst does with an unknown file is establish what it
actually is.

## Exercises

**B1.** Create a text file:

```bash
echo "This is ordinary text" > test.txt
```

**B2.** Ask what it is:

```bash
file test.txt
```

**B3.** Rename it with a misleading extension:

```bash
mv test.txt test.jpg
```

**B4.** Ask again:

```bash
file test.jpg
```

The name claims an image; `file` reports text. `file` is correct — it reads the first
bytes of the content, never the name. Every format begins with a recognisable signature
(*magic bytes*): `%PDF` for PDF, `FF D8 FF` for JPEG, `7F ELF` for a Linux executable.

**B5.** Character encoding also changes the answer:

```bash
echo "plain ascii text" > a.txt
file a.txt
echo "testo con è accentata" > b.txt
file b.txt
```

Two files that look the same, two different classifications. ASCII covers 128
characters and accented letters are not among them; they are represented in UTF-8 using
two bytes. One character changes the classification of the whole file.

This matters beyond curiosity: encoding mismatches are a constant source of trouble in
log analysis. A file exported as UTF-16 and read as UTF-8 produces garbage and looks
corrupted when it is not.

**B6.** Look at a real binary:

```bash
file /bin/ls
```

If the answer is `symbolic link`, follow it:

```bash
file -L /bin/ls
```

**B7.** Now open a binary as if it were text. **Expect the terminal to misbehave:**

```bash
cat /usr/bin/bash
```

**B8.** Repair the terminal:

```bash
reset
```

### What just happened, and why it is a security issue

`cat` sends bytes to the terminal without filtering them. The terminal interprets
certain byte sequences as *instructions* — change colour, move the cursor, report
identity — and a binary contains such sequences by coincidence.

In practice the terminal may respond to one of those instructions, and its response
arrives at the shell's **input**, as though it had been typed. Output like this is
normal after the exercise:

```
61;1;6;7;21;22;23;24;28;32;42;52c
61: command not found
```

Nobody typed those numbers. The terminal did.

This is a real attack class — *terminal escape injection*. A file supplied by someone
else can contain sequences that write into the reader's input, and in the worst case a
complete command followed by a newline.

**Working rule: files of uncertain origin are not opened with `cat`.** Use `less`,
which filters control sequences instead of executing them, or `strings`, which keeps
only printable characters.

**B9.** Extract the readable parts of a binary:

```bash
strings /bin/ls | head -20
```

`strings` scans the bytes and keeps only printable sequences of at least four
characters, discarding everything else. On a binary this surfaces function names, error
messages and paths — the readable traces a program carries inside it.

**B10.** Count them, then search within them:

```bash
strings /bin/ls | wc -l
strings /bin/ls | grep -i "usage"
```

## Verification

1. Why is `file` more reliable than an extension?
2. What does `strings` do, and why is it useful on a binary?
3. Given an attachment named `document.pdf`, which commands would you run before
   opening it?
4. What happens when `cat` is run on a binary, and why is it a security problem?
5. Why does one accented character change the output of `file`?

---

# Session C — Transforming and counting text

## Why it matters

A log is text. Counting occurrences, sorting by frequency, substituting characters:
these are the operations that turn ten thousand unreadable lines into an answer.

This session builds the pipeline used in the Phase 0 exit test.

## Part 1 — tr

**C1.** Simple substitution:

```bash
echo "abc" | tr 'abc' 'xyz'
```

`tr` aligns the two lists **by position**: first with first, second with second.

**C2.** Ranges instead of individual characters:

```bash
echo "hello world" | tr 'a-z' 'A-Z'
```

**C3.** Induce the mismatch error deliberately:

```bash
echo "abcdefg" | tr 'a-f' 'xyz'
```

Characters beyond the third all become `z`: when the second list is shorter, `tr` pads
by repeating its last character. The trailing `g` is untouched because it falls outside
the range `a-f`.

**C4.** Delete instead of substituting:

```bash
echo "h1e2l3l4o" | tr -d '0-9'
```

**C5.** Squeeze repeats:

```bash
echo "aaabbbccc" | tr -s 'abc'
```

## Part 2 — sort and uniq

**C6.** Create a file with scattered repeats:

```bash
printf 'dog\ncat\ndog\nmouse\ncat\ndog\n' > animals.txt
cat animals.txt
```

**C7.** Try `uniq` without sorting first:

```bash
uniq animals.txt
```

Nothing is removed. **`uniq` only compares adjacent lines**, and here the repeats are
scattered.

**C8.** Sort first:

```bash
sort animals.txt
```

**C9.** Combine:

```bash
sort animals.txt | uniq
```

**C10.** Count occurrences:

```bash
sort animals.txt | uniq -c
```

**C11.** Show only lines appearing exactly once:

```bash
sort animals.txt | uniq -u
```

**C12.** Order by frequency, highest first:

```bash
sort animals.txt | uniq -c | sort -rn
```

### Understanding the pipeline rather than memorising it

Three commands, three distinct jobs:

| Command | Job | Why it must be there |
|---|---|---|
| `sort` | Brings identical lines together | `uniq` only sees adjacent lines |
| `uniq -c` | Collapses and counts | Turns a list into count/value pairs |
| `sort -rn` | Reorders by that count | `-n` reads numbers, `-r` puts the largest first |

**C13.** Remove each stage in turn and observe what breaks:

```bash
uniq -c animals.txt | sort -rn        # counts are wrong: nothing was grouped
sort animals.txt | sort -rn           # no counts: nothing counted them
sort animals.txt | uniq -c            # counts, but in alphabetical order
```

This is not a formula to remember. It is three questions in sequence: *group*, *count*,
*rank*. If the data is already sorted, drop the first stage. If ordering does not
matter, drop the third.

The shape is not specific to `sort` and `uniq` — it is the Unix model. Small programs
that each do one thing, joined by pipes.

## Part 3 — numeric versus alphabetical sorting

**C14.** With single-digit counts the difference is invisible. Build a case where it
is not:

```bash
printf '9\n10\n2\n100\n' > numbers.txt
sort numbers.txt
sort -n numbers.txt
```

Without `-n`, sorting compares character by character, so `10` and `100` precede `2`.

**This matters directly.** When counting authentication failures, an address with 100
attempts sorted alphabetically lands below one with 9.

## Verification

1. Why does `uniq` alone often appear to do nothing?
2. What happens when `tr` receives lists of different lengths?
3. What is the difference between `sort -r` and `sort -rn`, and when is it visible?
4. Write from memory the pipeline that orders a file's lines by frequency.
5. Explain what each stage of that pipeline contributes.

---

# Session D — Compression and archives

## Why it matters

Files that arrive from elsewhere — exported logs, samples, archives — are usually
compressed, often more than once. Recognising the format and choosing the right tool is
mechanical work, but being unable to do it is a hard stop.

## Exercises

**D1.** Create and compress:

```bash
echo "sample content" > doc.txt
gzip doc.txt
ls
```

The original is gone: it became `doc.txt.gz`.

**D2.** Confirm the format and decompress:

```bash
file doc.txt.gz
gzip -d doc.txt.gz
```

**D3.** Repeat with bzip2. It may not be installed — Ubuntu Server no longer ships it
by default:

```bash
bzip2 doc.txt
```

If the command is missing, install it as the system suggests, then continue:

```bash
file doc.txt.bz2
bzip2 -d doc.txt.bz2
```

Worth noting: **available tools differ between machines.** A command present on this
lab may be absent on a system under analysis.

**D4.** Induce the extension error:

```bash
cp doc.txt copy
gzip copy
mv copy.gz noextension
gzip -d noextension
```

It refuses:

```
gzip: noextension: unknown suffix -- ignored
```

The message says *suffix*, not *format*. `gzip` recognises the format from the leading
bytes without help; it needs the extension to decide **what to call the decompressed
file**. It strips `.gz` and uses what remains. With nothing to strip, it stops.

**D5.** Restore the extension and retry:

```bash
mv noextension noextension.gz
gzip -d noextension.gz
```

**D6.** Archives behave differently. Create two files and group them:

```bash
echo "first" > one.txt
echo "second" > two.txt
tar cf archive.tar one.txt two.txt
```

**D7.** Inspect the contents without extracting:

```bash
tar tf archive.tar
```

**D8.** Delete the originals and extract:

```bash
rm one.txt two.txt
tar xf archive.tar
ls
```

The files return under their original names. `tar` does not compress — it groups, and
the names travel **inside** the archive. This is why `tar` needs no extension and
`gzip` does, and why `.tar.gz` exists: group first, then compress the bundle.

## Working method for layered files

A file compressed several times over is handled by repeating one cycle:

1. `file` — establish what this actually is
2. Rename if the tool requires a suffix
3. Decompress or extract with the matching tool
4. Delete the previous layer, so the working directory does not fill with intermediates
5. Return to step 1 until `file` reports text

Step 4 is the one most often skipped. After six or seven layers without it, the
directory holds fifteen files and no indication of which is current.

## Verification

1. What is the difference between `gzip` and `tar`?
2. Why does `gzip` require the correct extension when `tar` does not?
3. After `tar xf`, how do you know what the extracted files are called?
4. Describe the cycle for a file compressed through several unknown layers.

---

# Session E — Network services

## Why it matters

The previous sessions worked on files. This one works on services: programs listening
on a port, waiting to be spoken to.

These tools carry directly into later phases — verifying what a machine exposes, and
testing whether a firewall rule blocks what it claims to block.

## Part 1 — What is listening

**E1.** List listening services:

```bash
ss -tulpn
```

The flags: `t` TCP, `u` UDP, `l` listening only, `p` owning process, `n` numeric ports.

**E2.** Drop the `n` and compare:

```bash
ss -tulp
```

Port numbers become service names.

**E3.** The Process column is empty. Run it with privileges:

```bash
sudo ss -tulpn
```

The owning process now appears. **The same command returns different information
depending on who runs it** — without privileges the kernel withholds process ownership.

## Part 2 — nc

**E4.** Open a second session to the lab machine. Two concurrent terminals are needed.

**E5.** In the first, listen on a port:

```bash
nc -l 4444
```

It appears to hang. It is waiting.

**E6.** In the second, connect:

```bash
nc localhost 4444
```

**E7.** Type in either window and press Enter. The text appears in the other.

That is all `nc` does: it opens a TCP or UDP connection and passes bytes in both
directions. Chat, file transfer and port testing are all uses of the same mechanism.

**E8.** Close both with `Ctrl+C`.

**E9.** Test whether a port accepts connections:

```bash
nc -zv localhost 22
nc -zv localhost 9999
```

One reports `succeeded`, the other `refused`. This is the simplest possible check for
whether a service is answering.

## Part 3 — TLS

**E10.** `nc` sends bytes in the clear and cannot negotiate TLS. For an encrypted
service:

```bash
openssl s_client -connect ubuntu.com:443
```

**E11.** Read the output before typing anything. Four things are worth finding:

- `Certificate chain` — who vouches for this server
- `Protocol` — which TLS version was negotiated
- `Cipher` — which algorithm
- `Verify return code` — whether the certificate validated

**E12.** Close with `Ctrl+C`.

**E13.** Extract just the summary lines:

```bash
echo | openssl s_client -connect ubuntu.com:443 2>/dev/null | grep -iE "protocol|cipher|verify"
```

Three earlier concepts combined: `echo |` supplies immediate end-of-input, `2>/dev/null`
removes the noise, `grep` filters.

**The `-i` matters.** `grep` is case-sensitive by default, and the output contains
`Verify` with a capital V. Searching for `verify` without `-i` silently omits that line —
no error, just a missing result. The Session A problem again.

### Reading the verification result

A valid chain looks like this:

```
depth=0  CN=ubuntu.com
depth=1  Let's Encrypt
depth=2  ISRG Root YR
depth=3  ISRG Root X1
Verify return code: 0 (ok)
```

Each certificate is signed by the one above it, and the topmost is already present in
the machine's trust store. The chain terminates in something already trusted.

A self-signed certificate produces:

```
Verify return code: 18 (self-signed certificate)
```

**This is the distinction that matters.** TLS provides two separate guarantees:

| | Encryption | Identity |
|---|---|---|
| Valid certificate chain | yes | yes |
| Self-signed certificate | **yes** | **no** |

A self-signed certificate encrypts perfectly well. It does not establish *who* is on
the other end. An interceptor can present their own self-signed certificate and the
client cannot tell the difference.

This is what a browser warning actually reports — not "this connection is unencrypted"
but "I cannot confirm who this is".

## Verification

1. Describe what `nc` does in one sentence.
2. Why is `nc` insufficient for an HTTPS service?
3. What does `Verify return code: 18` mean, and what does it not mean?
4. Why does `ss -tulpn` show different information under `sudo`?
5. What is the difference between `localhost` and the machine's network address?

---

# Closing

```bash
cd ~
rm -r ~/exercises
```

Then reread every verification question in sequence. The ones answered without
hesitation are settled. The others indicate where to return.

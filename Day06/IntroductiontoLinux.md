# Linux & the Command Line — Study Notes

*Notes from Day 1 of my Operating System Security module*

## Why this matters
Every diagnosis I make later — in Wireshark, in a log analyzer, in an
incident response — eventually comes down to typing the right command at
a shell prompt. This is the layer underneath everything else I'm
learning. Three questions run through almost this whole session: *where
am I, what is here, and what is this actually — not what does it claim to
be.*

---

## Part 1: Getting my bearings

| Command | What it tells me |
|---|---|
| `pwd` | Print working directory — where I'm currently standing |
| `ls` / `ls -l` / `ls -a` / `ls -lah` | List files; long listing (permissions, owner, size, date); include hidden files; all combined with human-readable sizes |
| `ls -ld DIR` | The directory's own permissions, not what's inside it |
| `ls -lt` / `ls -ltr` | Sort by modified time, newest-first / oldest-first |
| `cd DIR`, `cd ..`, `cd ~`, `cd -` | Move around; up one level; home; previous folder |
| `file FILE` | What a file **actually is**, by reading its contents — not trusting its extension |
| `stat FILE` | Full metadata: exact permissions, owner, size, and access/modify/change timestamps |
| `which CMD` | Which file on disk actually runs when I type a command |
| `man CMD` / `CMD --help` | Full manual vs. quick reminder |
| `history` | Everything I've typed this session |

**The idea that stuck with me most here**: `file` reads content, not the
name. A file called `photo.jpg` that's secretly an executable can't hide
from `file` the way it can hide from a person just glancing at the
filename. This is my first real "don't trust what something claims to
be" lesson — a theme that repeats through the rest of the sheet.

**Two habits worth building now, not later:**
- **Look before I run.** `file` and `strings` inspect a file without
  executing it. Opening a suspicious file directly is how people
  compromise their own machine.
- **Let Tab do the spelling check.** If a path doesn't autocomplete, it
  doesn't exist — that's immediate, free feedback.

---

## Part 2: Reading files without opening them blind

| Command | Use |
|---|---|
| `cat FILE` | Dump the whole file — fine for short files only |
| `less FILE` | Page through — the right tool for logs (`/word` searches, `G` jumps to end) |
| `head -n 20 FILE` | First 20 lines — quick format check |
| `tail -n 50 FILE` | Last 50 lines — the recent activity in a log |
| `tail -f FILE` | Follow a file live as new lines are written |
| `wc -l FILE` | Count lines — often the second half of "how many failed logins," piped from a `grep` |
| `strings FILE` | Pull readable text out of a binary — a safe first look at something suspicious, since it only reads |
| `diff A B` | Show what differs between two files |
| `sha256sum FILE` | Fingerprint a file — same content always gives the same hash, proving nothing's been altered |
| `du -sh DIR` / `df -h` | Folder size / disk space. A full disk breaks logging — which then hides everything else that goes wrong |

**Connecting this to my Wireshark notes**: `sha256sum` here is the exact
same integrity-sealing idea as sealing a `.pcapng` file in the forensic
chain-of-custody workflow. Same tool, same underlying reason — prove a
file hasn't changed since I captured/found it.

---

## Part 3: Searching — grep looks *inside* files, find looks *for* files
This distinction is the whole point of this section, and it's worth
keeping completely straight: **grep** searches content; **find** searches
the filesystem by file properties (name, size, owner, modified time).

### grep — text inside files
```bash
grep 'WORD' FILE        # every line containing WORD
grep -i 'WORD' FILE     # case-insensitive
grep -n 'WORD' FILE     # show line numbers
grep -r 'WORD' DIR      # recursive, every file under a folder
grep -v 'WORD' FILE     # invert — lines that do NOT match (strip known-good noise)
grep -c 'WORD' FILE     # count matches instead of printing them
grep -w 'WORD' FILE     # whole word only (root doesn't also match chroot)
grep -E 'A|B' FILE      # match either pattern
grep -A 3 -B 3 'W' FILE # 3 lines of context after/before each hit
```

### find — files by their properties
```bash
find DIR -name '*.log'                          # by name
find DIR -iname '*.LOG'                         # by name, case-insensitive
find DIR -type f / -type d                      # files only / directories only
find DIR -user NAME                             # everything one account owns
find DIR -mmin -60                              # modified in the last 60 minutes
find DIR -size +100M                            # larger than 100MB (staged archives)
find / -perm -4000 -type f 2>/dev/null          # every SUID program on the system
find DIR -name 'X' -exec ls -l {} \;            # run a command on each result
```

**Why `2>/dev/null` shows up constantly**: scanning `/` as a normal user
throws a stream of "permission denied" noise. Throwing that away isn't
hiding anything important — it's just letting the real results surface.

**The single most important line in this whole section**, in my own
words: `find / -perm -4000 -type f 2>/dev/null` lists every program that
runs with someone else's permissions baked in (explained properly in Part
5). I need to actually understand *why* this matters before treating it
as just another command to memorize.

---

## Part 4: Pipes and redirection — small tools, joined together
The core idea: each command does one narrow thing well, and I chain them
together instead of looking for one command that does everything.

```bash
A | B              # pipe A's output straight into B
CMD > FILE         # write output to a file, REPLACING what was there
CMD >> FILE        # append instead — one character, and it's the
                   # difference between keeping and destroying my notes
CMD 2>/dev/null    # discard error messages, keep real output
CMD > FILE 2>&1    # send both output and errors to the same file
CMD | tee FILE     # show on screen AND save — how to keep evidence while working
```

**Shaping and summarizing output:**
```bash
sort / sort -u / sort -rn   # alphabetical / dedupe / highest number first
uniq -c                     # count repeats (input must be sorted first)
cut -d: -f1                 # split on : and keep field 1
awk '{print $1, $5}'        # print specific columns (splits on whitespace)
tr -d '\r'                  # delete characters (fixes Windows line endings)
xargs CMD                   # turn a list of lines into arguments for another command
```

**Worked example I want to be able to rebuild from memory:**
```bash
cat /etc/passwd | grep -v nologin | cut -d: -f1 | sort
```
Read left to right: read the file → drop service accounts → keep just
the username → alphabetize. This is the identical "small tools piped
together" pattern from my TShark notes
(`tshark ... | sort | uniq -c`) and from my own log analyzer project —
same philosophy, three different contexts now.

**The "top offenders" recipe** — genuinely one of the most reusable lines
on the whole sheet:
```bash
sudo grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head
```
Count → sort → rank, busiest first. I want to be able to explain every
piece of this without looking it up: grep finds the failed attempts, awk
pulls out the source IP field, sort groups identical IPs together (which
uniq -c requires), uniq -c counts them, sort -rn ranks by count
descending, head keeps just the top results.

---

## Part 5: Permissions — the ten characters

```
-rw-r--r-- 1 diya staff 4096 Aug 24 09:14 notes.txt
```

| Position | Example | Controls | Meaning |
|---|---|---|---|
| 1st | `-` | File type | `-` file, `d` directory, `l` symlink, `c`/`b` device, `s` socket, `p` pipe |
| 2nd–4th | `rw-` | Owner | Can read/write, not execute |
| 5th–7th | `r--` | Group | Read only |
| 8th–10th | `r--` | Others | Read only |

**The arithmetic — read=4, write=2, execute=1, added per audience:**

| Octal | Letters | Typical use |
|---|---|---|
| 7 | rwx | Owner of a script or directory |
| 6 | rw- | Owner of a plain data file |
| 5 | r-x | Others on a shared program/folder |
| 4 | r-- | Others on a public config file |
| 0 | --- | Others on anything sensitive |

So `644 = rw-r--r--`, `600 = rw-------`, `755 = rwxr-xr-x`,
`700 = rwx------`.

**A directory's letters mean something different from a file's** — this
genuinely surprised me: `r` = list the names inside it, `w` = create/
delete entries, `x` = actually enter it and reach what's inside. A
directory I can read but not execute (`x`) into is closed even if I can
technically see file names sitting in it — the names are visible, but
I can't step inside.

```bash
chmod 640 FILE           # numeric form
chmod u+x FILE           # symbolic form: add execute for owner
chmod -R 750 DIR         # recursive — powerful, easy to regret, check the path twice
chown USER:GROUP FILE    # change owner and group
umask                    # withheld permissions for new files (022 → 644/755, 077 → 600/700)
```

### The special bits — this is the part that actually matters for security
| Bit | Looks like | Octal | Why it's dangerous/useful |
|---|---|---|---|
| SUID | `-rwsr-xr-x` | 4755 | Program runs with the **owner's** power, not mine. If root owns it and it can be tricked into running something else, that's a path to a root shell — the classic privilege escalation route |
| SGID | `-rwxr-sr-x` | 2755 | Same idea for group; on a directory, new files inherit the folder's group — usually harmless on shared team folders |
| Sticky | `drwxrwxrwt` | 1777 | In a world-writable folder, only the file's *own* owner can delete it — this is literally why `/tmp` is usable at all |

**Connecting this back to Part 3**: this is *why* `find / -perm -4000
-type f 2>/dev/null` matters so much. It's not just "list some files" —
it's specifically hunting for every program capable of handing me
elevated power. The sheet says to expect a short, boring list (`passwd`,
`sudo`, `mount`, `su`) — anything unexpected, especially somewhere like
`/home` or `/tmp`, is a real red flag worth investigating.

---


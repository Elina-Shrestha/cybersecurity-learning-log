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

## Part 6: Users, groups, and accounts

| Command | Purpose |
|---|---|
| `whoami` / `id` | Who am I / full identity: UID, GID, all group memberships |
| `groups USER` | Just the group memberships |
| `cat /etc/passwd` | Every account — readable by all, despite the misleading name, no actual passwords in it |
| `sudo cat /etc/shadow` | Where password hashes actually live — root-only, and that restriction is the entire point |
| `awk -F: '$3==0 {print $1}' /etc/passwd` | Every account with UID 0 — should be exactly one (`root`); a second is a backdoor |
| `sudo -l` | What am I allowed to run as root — first command in any privilege review, and reportedly the first thing an attacker types too |
| `sudo visudo` | The only safe way to edit sudo rules — refuses to save a file that would lock everyone out |
| `sudo usermod -aG GROUP USER` | Add to a group — the `-a` (append) matters; without it, I'd silently wipe every other group membership |
| `sudo usermod -L` / `-U` | Lock / unlock an account — reversible, unlike deleting, which destroys evidence |

**The seven fields of `/etc/passwd`, decoded** (using the sheet's own
example):
```
diya : x : 1001 : 1001 : Diya,,, : /home/diya : /bin/bash
```
Login name → password placeholder (real hash is in `/etc/shadow`) → user
ID → primary group ID → description → home directory → login shell.

**One detail I want to remember specifically**: `/usr/sbin/nologin` as
the shell is normal for service accounts, but *suspicious if it suddenly
changes* — a service account that shouldn't be able to log in
interactively, quietly gaining a real shell, is exactly the kind of quiet
change worth noticing.

---

## Part 7: Processes — what's actually running

| Command | Purpose |
|---|---|
| `ps aux` | Every process: user, PID, CPU, memory, and the **full command line** — the command line is the actual evidence |
| `ps -ef --forest` / `pstree -p` | Parent-child relationships — "who launched this" answers more than "what is this" |
| `top` / `htop` | Live view, busiest first |
| `kill PID` | Ask a process to stop cleanly (signal 15) — always try this first |
| `kill -9 PID` | Force-kill (signal 9) — no cleanup, last resort |
| `ls -l /proc/PID/exe` | The *real* program behind a PID, even if its displayed name is lying |

**The idea I want to hold onto**: a process's displayed name is just a
label — `/proc/PID/exe` and the full command line in `ps aux` are what
actually tell the truth, the same "don't trust the label" theme as `file`
checking real content back in Part 1.

---

## Part 8: Services — what starts itself

| Command | Purpose |
|---|---|
| `systemctl status NAME` | Running? enabled at boot? recent log lines — the starting point |
| `systemctl list-units --type=service --state=running` | Everything currently running |
| `systemctl list-unit-files --state=enabled` | Everything set to start at boot, even if not running now |
| `sudo systemctl disable --now NAME` | Stop it now AND prevent it restarting — the actual hardening command |
| `sudo systemctl mask NAME` | Stronger than disable — makes it unstartable even by hand |
| `systemctl list-timers` | systemd's version of cron — a place attackers hide persistence, since fewer people check it |

**Important distinction I don't want to blur**: `stop` only ends
something until the next reboot; `disable` stops it coming back but
doesn't touch its current running state. Hardening needs both together —
which is exactly what `disable --now` does in one step.

---

## Part 9: Network — what's listening, and to whom

| Command | Purpose |
|---|---|
| `ip a` / `ip -br a` | My own addresses |
| `sudo ss -tulpn` | **The important one** — TCP/UDP, listening, process owner, numeric ports. Without `sudo`, I see the ports but not who owns them |
| `sudo ss -tp` | Established connections and their owning processes — who's talking to whom right now |
| `sudo lsof -i` | Same question, from the file-descriptor side — often clearer |
| `dig NAME` / `dig -x IP` | Name → address, and address → name |
| `sudo ufw status verbose` | Ubuntu's firewall status |

**The single detail from this whole section I most want to remember**:
`127.0.0.1:3306` only accepts connections from the machine itself.
`0.0.0.0:3306` accepts connections from anywhere that can route to this
host. That one difference in the address is often the entire security
finding — a database that should only ever be `127.0.0.1` suddenly
listening on `0.0.0.0` is a real exposure, not a cosmetic detail.

This connects directly back to my port scanner project — `ss -tulpn` is
essentially the "ground truth" answer that my own scanner is trying to
infer from the outside, without needing access to the machine itself.

---

## Part 10: Logs — where Linux writes down what happened

| Command | Purpose |
|---|---|
| `sudo journalctl -u SERVICE` | Everything one service has logged |
| `sudo journalctl -f` | Follow live, like `tail -f` |
| `sudo journalctl --since "1 hour ago"` | Time-bounded search |
| `sudo grep "Failed password" /var/log/auth.log` | Failed logins (Debian/Ubuntu; `/var/log/secure` on RHEL-family) |
| `sudo grep "Accepted" /var/log/auth.log` | Successful logins — method and source address |
| `last` / `sudo lastb` | Recent successful logins / recent **failed** logins (often more interesting) |

**Key log locations to remember:**

| Path | What's there |
|---|---|
| `/var/log/auth.log` | Logins, sudo, SSH, account changes (Debian/Ubuntu) |
| `/var/log/secure` | Same, on RHEL-family systems |
| `/var/log/syslog` | General system messages |
| `~/.bash_history` | Commands a user typed — easy to edit/delete, so **a lead, never proof** |

That last point matters: unlike a properly configured system log,
`.bash_history` is something the user themselves controls, so I shouldn't
treat it as reliable evidence the way I would `auth.log` or `journalctl`.

---

## Part 11: Patching and packages

```bash
apt list --upgradable      # what's out of date (safe, changes nothing)
sudo apt update            # refresh the catalogue (installs nothing)
sudo apt upgrade           # actually install updates
dpkg -S /path/to/file      # which package owns a file I didn't expect
cat /etc/os-release        # which distro/version this really is
uname -r                   # running kernel version (a patched kernel needs a reboot to take effect)
sudo lynis audit system    # a free automated hardening audit
```

**Small but important distinction**: `apt update` refreshes what's
*available*; it installs nothing by itself. Skipping it before `apt
upgrade` means patching from a stale list — an easy mistake to make by
assuming "update" and "upgrade" are basically the same word.

---

## Part 12: Five ways people break their own machine
Worth memorizing as warnings, not just facts:
1. `rm -rf PATH` — one typo or stray space, and it's gone, no recycle bin. Always run the exact path through `ls` first.
2. `chmod 777 FILE` — "fixes" nothing, hands full access to every account on the system.
3. `chmod -R` / `chown -R` aimed at `/` or `/etc` — does exactly what I told it, everywhere, instantly, irreversibly.
4. `nano /etc/sudoers` — one syntax error locks everyone out of sudo. Always use `visudo`, which refuses to save something broken.
5. `curl URL | sudo bash` — running unread code, as root, from a machine I don't control. Download it, read it, *then* run it.

---

## Part 13: A first-hour triage sequence (read-only, changes nothing)

This is the section I want to actually be able to run from memory one day:

| # | Command | The question it answers |
|---|---|---|
| 1 | `id ; sudo -l` | Who am I, what am I allowed to do — establish my own footing first |
| 2 | `who ; last \| head -20` | Who's on the box now/recently — odd hours or unrecognized addresses are the flag |
| 3 | `sudo lastb \| head -20` | Who's been failing to log in — a burst against one account is brute force; one failure each across many accounts is password spraying |
| 4 | `awk -F: '$3==0 {print $1}' /etc/passwd` | Is there more than one root-equivalent account? |
| 5 | `ps aux --sort=-%cpu \| head -15` | What's running, busiest first — read the full command line |
| 6 | `sudo ss -tulpn` | What's listening, owned by whom — anything on `0.0.0.0` I can't explain |
| 7 | `systemctl list-units --type=service --state=running` | What services are up — compare against a known-good build |
| 8 | `find / -perm -4000 -type f 2>/dev/null` | What runs with borrowed power |
| 9 | `find /etc /home -mmin -1440 -type f 2>/dev/null` | What's been edited in the last 24 hours |
| 10 | `sudo journalctl --since "24 hours ago" -p err` | What has the system complained about |

**Two rules that apply to the whole sequence:**
- **Record as I go** — pipe through `| tee ~/triage.txt` so the output,
  order, and timestamps are preserved afterward. Memory isn't evidence.
- **Change nothing yet** — every command above only *reads*. The moment
  I start killing processes or editing files, I'm destroying the record
  I was supposed to be collecting. This is the same principle as the
  forensic chain-of-custody notes from Wireshark — investigate first,
  act second.

---

## The six commands to have cold, no matter what
```
ls -lah
grep -rn
find / -perm -4000 -type f 2>/dev/null
ps aux
sudo ss -tulpn
sudo -l
```
Between them: what's here, what's inside it, what runs with borrowed
power, what's running, what's listening, and what am I allowed to do.

## What I'd still want to practice
- Actually running the full Part 13 triage sequence on a real VM,
  start to finish, timing myself — reading it is not the same as being
  able to reach for it under pressure
- Getting fast enough with `awk`/`cut` field extraction that I don't need
  to look up the syntax every single time — right now I understand *why*
  each worked example works, but couldn't yet write one from scratch
  without a reference
# Linux Command Cheat Sheet
### Operating System Security — Day 1
**Verified on Ubuntu 24.04 LTS**

---

## 01 — Getting Around

| Command | Description |
|---|---|
| `pwd` | Print working directory — the folder you are standing in. Type it whenever you are lost. |
| `ls` | List the files in this folder. |
| `ls -l` | Long listing: permissions, owner, group, size, date. The single most useful flag on the system. |
| `ls -a` | Show hidden files too. On Linux, "hidden" just means the name starts with a dot. |
| `ls -lah` | All three combined, with human-readable sizes (`4.0K` instead of `4096`). The habit to build. |
| `ls -ld DIR` | Show the directory itself, not what is inside it. Needed when you care about the folder's own permissions. |
| `ls -lt` / `ls -ltr` | Sort by modification time, newest first / oldest first. In an investigation, "what changed last?" is often the whole question. |
| `cd DIR` | Change directory. |
| `cd ..` / `cd ~` | Up one level / to your home folder. |
| `cd -` | Back to the previous folder. |
| `file FILE` | What kind of file is this really? Reads the contents, not the extension — so a `photo.jpg` that is actually a program cannot hide. |
| `stat FILE` | Full metadata: exact permissions, owner, size, and the access / modify / change timestamps. |
| `which CMD` | Which file on disk actually runs when you type that command. Useful when something behaves oddly. |
| `man CMD` | The manual. `q` quits, `/word` searches, `n` jumps to the next match. |
| `CMD --help` | The short version. Faster than the manual when you only forgot one flag. |
| `history` | Every command you have typed. A shell builtin, so it has no manual page of its own — see `man bash`. |
| `clear` | Wipe the screen. Same as `Ctrl+L`. |

---

## 02 — Reading Files

| Command | Description |
|---|---|
| `cat FILE` | Dump the whole file to the screen. Fine for short files; painful for long ones. |
| `less FILE` | Page through a file. Arrows scroll, `/word` searches, `G` jumps to the end, `q` quits. Use this for logs. |
| `head -n 20 FILE` | First 20 lines. Good for checking a file's format. |
| `tail -n 50 FILE` | Last 50 lines. In logs, the last lines are the recent ones. |
| `tail -f FILE` | Follow the file live — new lines appear as they are written. Run this, then trigger the event, and watch it land. `Ctrl+C` stops. |
| `wc -l FILE` | Count lines. "How many failed logins?" is usually a `grep` piped into `wc -l`. |
| `nl FILE` | Show the file with line numbers. |
| `strings FILE` | Pull readable text out of a binary. The safe first look at a suspicious program — it reads, it never runs. |
| `diff A B` | Show what differs between two files. Compare a config against a known-good copy. |
| `sha256sum FILE` | Fingerprint a file. Same contents give the same hash every time, so it proves a file has not been altered. |
| `du -sh DIR` | How big is this folder, total. |
| `df -h` | How full are the disks. A disk at 100% breaks logging — and broken logging hides everything else. |

### Two Habits That Keep You Out of Trouble

> **Look before you run.** `file` and `strings` inspect a file without executing it. Double-clicking a sample is how analysts infect their own machines.

> **Press `Tab` to finish a name.** If it does not complete, the path you typed does not exist — that is a free spelling check on every command.

---

## 03 — Searching — Inside Files, and for Files

### GREP — Find Text Inside Files

| Command | Description |
|---|---|
| `grep 'WORD' FILE` | Print every line containing that word. |
| `grep -i 'WORD' FILE` | Ignore upper/lower case. Attackers do not capitalise consistently; neither should your search. |
| `grep -n 'WORD' FILE` | Show line numbers with each match. |
| `grep -r 'WORD' DIR` | Search every file under a folder, recursively. |
| `grep -v 'WORD' FILE` | Invert — show lines that do not match. The fastest way to strip known-good noise out of a log. |
| `grep -c 'WORD' FILE` | Count matching lines instead of printing them. |
| `grep -w 'WORD' FILE` | Whole word only, so `root` does not also match `chroot`. |
| `grep -E 'A\|B' FILE` | Extended pattern — match either one. Combine several searches into one pass. |
| `grep -A 3 -B 3 'W' FILE` | Show 3 lines **After** and **Before** each hit. Context is usually where the answer is. |

### FIND — Locate Files by Their Properties

| Command | Description |
|---|---|
| `find DIR -name '*.log'` | By name. Quote the pattern so the shell does not expand it first. |
| `find DIR -iname '*.LOG'` | By name, ignoring case. |
| `find DIR -type f` / `-type d` | Only regular files / only directories. |
| `find DIR -user NAME` | Everything owned by one account. |
| `find DIR -mmin -60` | Modified in the last 60 minutes. `-mtime -1` is the last 24 hours. The go-to command after "something happened this morning". |
| `find DIR -size +100M` | Larger than 100 megabytes. Finds staged archives an attacker forgot to delete. |
| `find / -perm -4000 -type f 2>/dev/null` | Every SUID program on the system. Read section 05 before you run this — it is the single most important privilege escalation check. |
| `find DIR -name 'X' -exec ls -l {} \;` | Run a command on each result. `{}` is the filename; `\;` ends it. |
| `... 2>/dev/null` | Throw away the "Permission denied" complaints so you can see the actual results. Add it to almost every `find /`. |

---

## 04 — Pipes, Redirection, and Shaping Output

### Small Tools, Joined Together

The idea: each command does one thing and passes its output to the next.

```bash
cat /etc/passwd | grep -v nologin | cut -d: -f1 | sort
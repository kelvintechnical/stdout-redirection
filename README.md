# Lab: Standard Output Redirection — `>`, `>>`, `cat`

**Series:** linux-ops-mastery — RHCSA Shells, Terminals & Redirection
**Subjects covered:** File-descriptor model (FD 0/1/2), `stdout` (FD 1), the truncate-on-write `>` operator, the append `>>` operator, `cat` as a stream tool, `/dev/null`, `set -o noclobber`, redirection-order rules, exit status preservation through redirection
**Career arcs covered:** RHCSA (every "save the output to a file" sub-task on EX200), RHCE (Ansible `command:` / `shell:` modules capture stdout into `result.stdout`), SRE (capturing incident command output without overwriting prior evidence), DevOps (CI/CD jobs piping build logs into artifact files), AI/MLOps (training-script stdout capture into experiment logs)
**Prerequisite:** Basic shell familiarity — you can `ls`, `pwd`, `cat`, and you know what a file path is
**Time Estimate:** 30 to 45 minutes
**Difficulty arc:** Task 1 foundation · 2–3 the `>` vs `>>` core split · 4–5 chaining, safety nets, `/dev/null` · 6 RHCSA exam-realistic capstone

---

## Objective

Make output redirection a reflex. By the end of this lab you will never again miss an answer because the data you needed was scrolling off the screen instead of saved to disk. Every RHCSA task that says *"save the output to..."* or *"capture the contents of..."* reduces to a `>` or `>>` decision and a verification step. You will own both decisions.

The capstone is an exam-realistic prompt: *"Save the output of `ps -ef` to `/root/ps.txt`, then append a timestamped header line to the same file without losing the existing content."*

> **Lab safety note:** Every command in this lab writes to files under `/tmp/redir-lab` or your home directory. Nothing touches a system file. The `set -o noclobber` section shows you the muscle memory that prevents a misplaced `>` from destroying a config file in real life.

---

## Concept: stdout Is a Stream, Not a Screen

When a command "prints to the screen," what actually happens is the kernel writes bytes to **file descriptor 1** of the process. The terminal happens to be connected to FD 1 by default, but FD 1 is a *handle* — point it at a file and the bytes land in the file instead.

```
   ┌─────────────────────────────────────────────────────┐
   │   Your command (ls, ps, cat, awk, find, ...)        │
   ├─────────────────────────────────────────────────────┤
   │   FD 0  stdin   ← keyboard (default)                │
   │   FD 1  stdout  → terminal screen (default)         │  ← `>` and `>>` redirect THIS
   │   FD 2  stderr  → terminal screen (default)         │
   └─────────────────────────────────────────────────────┘
                                  │
                  ┌───────────────┴───────────────┐
                  │                               │
            ` > file`                       ` >> file`
       truncate then write              create if missing, append
       (destroys existing content)      (preserves existing content)
```

Every modern Linux command obeys this convention. That's why `ls > out` works and `find / > out` works and `awk -f script.awk input > out` works — they all just write to FD 1.

> **Why this matters:** On the RHCSA exam, the graders run a script that checks file contents. If your "answer" was only on screen and the file is empty, you get zero points — even though you typed the right command.

---

## 📜 Why Output Redirection Exists — The Story

In 1969, when Ken Thompson and Dennis Ritchie were sketching what would become Unix on a PDP-7 at Bell Labs, every other operating system in existence treated each program's I/O as a custom problem. To read a card deck, you wrote card-reading code. To print to a line printer, you wrote line-printer code. Each new device meant new code.

Thompson's insight was brutal in its simplicity: **everything is a file.** Programs don't read "from a card reader" — they read from FD 0 and don't care what's behind it. Programs don't write "to a line printer" — they write to FD 1 and don't care what's behind it. The shell can swap whatever it wants behind those numbers before the program starts.

This single decision created:

- **Redirection** (`>`, `>>`, `<`) — swap a file in for FD 0 or FD 1 without the program knowing.
- **Pipes** (`|`) — connect FD 1 of one program to FD 0 of another, no temp file needed.
- **Tee fittings** (`tee`) — clone the stream to a file while still passing it on.
- **The whole Unix philosophy** of small, composable tools that read from stdin and write to stdout.

Fifty-seven years later, that decision is why a Bash one-liner can do what required a 200-line C program before Unix. The `>` operator is older than the C programming language, older than email, and older than every operating system you have ever used. Master it and you have mastered the foundational primitive of every shell that has shipped since 1971.

> **The point of the story:** `>` and `>>` are not "shell tricks." They are the original design of how a Unix program talks to the outside world. Every container log driver, every systemd `StandardOutput=`, every Ansible `register:` value is a descendant of this 1969 idea.

---

## 👪 The Redirection Family — Who Lives There

Standard-output redirection is one branch of a much larger family. Knowing the whole family helps you choose the right operator instantly.

### By direction

| Operator | What moves | Default file behavior |
|---|---|---|
| `>` | stdout (FD 1) → file | **Truncate** if exists, create if missing |
| `>>` | stdout (FD 1) → file | **Append**, create if missing |
| `2>` | stderr (FD 2) → file | Truncate if exists, create if missing |
| `2>>` | stderr (FD 2) → file | Append, create if missing |
| `<` | file → stdin (FD 0) | Read-only source |
| `<<EOF` | inline heredoc → stdin | Multi-line literal input |
| `&>` | stdout + stderr → file | Bash shorthand for `> file 2>&1` |
| `\|` | stdout → next command's stdin | No file involved |

### By safety habit

| Pattern | What it does | When to reach for it |
|---|---|---|
| `cmd > file` | Overwrite | First write of a fresh file |
| `cmd >> file` | Append | Logs, incident notes, accumulating output |
| `cmd > /dev/null` | Discard | Suppress noisy stdout |
| `cmd \| tee file` | Display **and** save | When you want to see it scroll *and* keep a record |
| `cmd \| tee -a file` | Display **and** append | Same, but preserves prior file content |
| `set -o noclobber` then `cmd > file` | Refuses if `file` exists | Safety net against fat-fingered overwrites |
| `cmd >\| file` | Force overwrite even under noclobber | Explicit "yes, clobber it" override |

### By what `cat` adds to the mix

| Pattern | Use case |
|---|---|
| `cat FILE` | Print one file to stdout (then optionally redirect) |
| `cat F1 F2 F3 > combined.txt` | Concatenate multiple files into one |
| `cat > newfile.txt` (then Ctrl-D) | Capture typed input into a file |
| `cat <<'EOF' > config.conf` | Script-friendly multi-line file creation |
| `cat FILE \| cmd` | Useless use of cat (UUOC) — prefer `cmd < FILE` |

> **The point of the family tree:** `>`, `>>`, and `cat` are three tools for three jobs. `>` writes fresh files. `>>` accumulates output. `cat` joins streams together. The instant you can name the job, the operator picks itself.

---

## 🔬 The Anatomy of a Redirection — In One Diagram

```
$ ps -ef > /root/ps.txt
  │   │  │      │
  │   │  │      └─ Target file. Created if missing; truncated if it exists.
  │   │  └─ The `>` operator: "send FD 1 of the command on the left to the path on the right."
  │   └─ Command-line arguments to the command.
  └─ The command whose stdout we are redirecting.

What the kernel actually does:
  1. Fork a child process for `ps`.
  2. BEFORE exec'ing `ps`, the shell `open(2)`s /root/ps.txt with O_WRONLY|O_CREAT|O_TRUNC.
  3. The shell `dup2(2)`s that file's FD onto FD 1.
  4. exec(`ps`, `-ef`). Now `ps` writes to FD 1 as usual — but FD 1 is the file.
  5. When `ps` exits, the kernel closes the file. The shell sees the exit code.

Append (`>>`) is identical except the file is opened with O_WRONLY|O_CREAT|O_APPEND.
```

> **Reading rule:** Redirection is set up by the **shell**, before the command runs. The command itself never knows it is writing to a file. That's why every command on Linux supports `>` and `>>` for free — there's nothing for the command author to implement.

---

## 📚 Redirection Reference Table

| Task | Command | Notes |
|---|---|---|
| Save stdout to a new file | `cmd > file` | Overwrites if `file` exists |
| Append stdout to a file | `cmd >> file` | Creates `file` if missing |
| Discard stdout | `cmd > /dev/null` | The "bit bucket" — bytes go nowhere |
| Print and save | `cmd \| tee file` | Screen plus file (overwrite mode) |
| Print and append | `cmd \| tee -a file` | Screen plus file (append mode) |
| Combine many files | `cat f1 f2 f3 > all` | Files joined in argument order |
| Create a file from typed input | `cat > note.txt` then Ctrl-D | Useful in pinch when no editor is handy |
| Multi-line file from script | `cat <<'EOF' > unit.service ... EOF` | Heredoc — single-quoted EOF disables `$var` expansion |
| Protect against accidental clobber | `set -o noclobber` | Then `>` refuses to overwrite existing files |
| Force overwrite under noclobber | `cmd >\| file` | Explicit override |
| Save the exit code, even when redirecting | `cmd > file; echo $?` | The redirection does not change `$?` |

> **Rule one of `>` vs `>>`:** Ask yourself, *"if this file already has data in it, do I want it gone?"* Yes → `>`. No → `>>`. That single question prevents the single most common data-loss accident in a shell.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | EX200 graders inspect files, not screens. Every "save the output" prompt requires `>` (or `>>`) plus a verification `cat`. |
| **RHCE candidate** | Ansible's `command:`, `shell:`, and `script:` modules return `result.stdout` — that value is exactly what `>` would have captured. Same mental model. |
| **SRE / Platform** | Incident response: run `ss -tunap > /tmp/incident-$(date +%s)-sockets.txt` to freeze evidence. `>>` keeps a running incident log without losing earlier captures. |
| **DevOps** | Every CI/CD step that "uploads logs as artifacts" is `>` under the hood — `npm test > test.log` then attach `test.log`. |
| **AI / MLOps** | Training runs print learning-rate / loss / accuracy to stdout; `python train.py > runs/exp-42.log` is how you save experiments for later analysis. |

---

## 🔧 The 6 Tasks

> This lab is grouped into six exam-realistic phases so the **stdout → file → verify** habit is easy to read, rehearse, and memorize.

---

### Task 1 — Set up the sandbox and inspect default stdout

**Purpose:** Create a clean working directory, confirm the three file descriptors are present, and prove what "default stdout" looks like before you redirect anything.

```bash
mkdir -p /tmp/redir-lab && cd /tmp/redir-lab

ls /proc/self/fd
echo "hello" 
echo "hello to file" > hello.txt

ls -l hello.txt
cat hello.txt
```

**Human-Readable Breakdown:** Make a sandbox directory, peek at the current process's open file descriptors (you'll see 0/1/2), print a line normally so you see it land on the terminal, then perform your first `>` redirect and read the file back.

**Reading it left to right:** `mkdir -p` creates the directory if it does not exist. `cd` enters it. `ls /proc/self/fd` shows the kernel-view of FDs 0, 1, 2 (and 3 — the `ls` command itself reading the directory). `echo "hello"` prints to FD 1 → terminal. `echo "hello to file" > hello.txt` opens `hello.txt`, points FD 1 at it, then runs `echo`. `cat hello.txt` proves the bytes landed.

**The story:** Before you can trust redirection, prove to yourself it works on a single line you can verify. Every senior engineer's first redirection in a new shell is `echo test > /tmp/test && cat /tmp/test` — it takes two seconds and confirms the shell, the terminal, and the filesystem are healthy.

**Expected output:**

```text
0  1  2  3
hello
-rw-r--r--. 1 user user 14 May 26 13:00 hello.txt
hello to file
```

**Switches**

| Token | Meaning |
|---|---|
| `mkdir -p PATH` | Create the path, do nothing if it already exists |
| `ls /proc/self/fd` | Show open file descriptors for the current process |
| `echo "TEXT"` | Print TEXT followed by a newline to stdout |
| `> file` | Send stdout to `file`, truncating if it exists |
| `cat file` | Print the file's bytes to stdout |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `bash: hello.txt: Permission denied` | You are in a directory you cannot write to — `cd /tmp/redir-lab` first |
| `> hello.txt: No such file or directory` | The parent directory does not exist — `mkdir -p` it |
| `ls /proc/self/fd` lists more than 0/1/2 | Normal — extra FDs are opened by the listing command itself |

---

### Task 2 — Overwrite with `>` and observe truncate-on-write

**Purpose:** Use `>` to capture command output into a new file, then deliberately re-run with `>` to watch the file get truncated and replaced. Truncation is the most common cause of accidental data loss in a shell — see it once, never forget it.

```bash
cd /tmp/redir-lab

date > stamp.txt
cat stamp.txt
wc -l stamp.txt

ls /etc > etc-list.txt
wc -l etc-list.txt
head -n 3 etc-list.txt

ls /usr/bin > etc-list.txt
wc -l etc-list.txt
head -n 3 etc-list.txt
```

**Human-Readable Breakdown:** Stamp the current time into `stamp.txt`. Capture the list of `/etc` into `etc-list.txt`. Then deliberately re-redirect a different listing into the same file — observe that the original `/etc` listing is gone.

**Reading it left to right:** `date > stamp.txt` writes a single line. `ls /etc > etc-list.txt` writes ~200 lines. `ls /usr/bin > etc-list.txt` opens `etc-list.txt` with `O_TRUNC` (length set to zero) **before** `ls` even runs, then writes ~2000 different lines. The first capture is gone forever.

**The story:** `>` does not "add to" a file. It empties the file, then writes. If you wanted to keep the old content, you needed `>>`. This single misunderstanding has destroyed config files, log archives, and home directories worth of work since 1971.

**Expected output:**

```text
Tue May 26 13:01:14 EDT 2026
1 stamp.txt
234 etc-list.txt
adjtime
aliases
alsa
2879 etc-list.txt
2to3
2to3-3.9
411toppm
```

**Switches**

| Token | Meaning |
|---|---|
| `date` | Print current date and time to stdout |
| `wc -l FILE` | Count lines in FILE |
| `head -n 3 FILE` | Print the first 3 lines of FILE |
| `> FILE` | Send stdout to FILE, truncating it first |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| The file is empty after the command | The command wrote to stderr, not stdout — use `&>` (covered in Lab 04) |
| You needed the old content and now it's gone | Recover from backup; next time use `>>` if accumulating |
| `wc -l` shows 0 lines but the file is not empty | The last line lacks a trailing newline — file still has content |

---

### Task 3 — Append with `>>` and accumulate output

**Purpose:** Use `>>` to add new lines to an existing file without destroying prior content. This is the operator behind every log file, every running incident report, and every "keep adding to this list" workflow.

```bash
cd /tmp/redir-lab

echo "First entry  $(date -Is)" > journal.txt
echo "Second entry $(date -Is)" >> journal.txt
echo "Third entry  $(date -Is)" >> journal.txt

cat journal.txt
wc -l journal.txt

uname -a >> journal.txt
whoami   >> journal.txt
cat journal.txt
```

**Human-Readable Breakdown:** Create `journal.txt` with one initial line using `>`. Append three more lines with `>>`. Notice that the first line is preserved every time. Then append the kernel version and the current user — anything that writes to stdout can be `>>`'d.

**Reading it left to right:** First line uses `>` to **create** `journal.txt`. Each subsequent `>>` opens the same file with `O_APPEND`, so the kernel always writes at the current end-of-file. The file grows; nothing is overwritten. `cat` and `wc -l` confirm.

**The story:** `>>` is the operator that turns single-line commands into running logs. Every `/var/log/*.log` file you have ever read was assembled by `>>` (or its programmatic equivalent, `O_APPEND`). When in doubt about whether to use `>` or `>>` for a file you might want to keep — choose `>>`.

**Expected output:**

```text
First entry  2026-05-26T13:02:00-04:00
Second entry 2026-05-26T13:02:00-04:00
Third entry  2026-05-26T13:02:00-04:00
3 journal.txt
First entry  2026-05-26T13:02:00-04:00
Second entry 2026-05-26T13:02:00-04:00
Third entry  2026-05-26T13:02:00-04:00
Linux ip-10-0-0-12 5.14.0-427.13.1.el9_4.x86_64 #1 SMP ...
ec2-user
```

**Switches**

| Token | Meaning |
|---|---|
| `>> FILE` | Send stdout to FILE in append mode, create if missing |
| `date -Is` | ISO-8601 timestamp with seconds (sortable) |
| `uname -a` | Print kernel name, host, version, architecture |
| `whoami` | Print the current effective username |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `>>` wrote nothing and the file is unchanged | The command itself produced no stdout — check by running it without redirection |
| Permission denied on append | You can `>>` only files you have write permission on — `chmod` or `sudo` |
| Lines are out of order | `>>` writes at end-of-file at the moment of each open; if two writers race, results interleave |

---

### Task 4 — Combine `cat`, multiple files, and `tee`

**Purpose:** Use `cat` to concatenate multiple files into one and `tee` to display output on screen while also saving it. These are the two patterns that turn `>` and `>>` from "write one file" into a real toolkit.

```bash
cd /tmp/redir-lab

echo "alpha"   > part1.txt
echo "bravo"   > part2.txt
echo "charlie" > part3.txt

cat part1.txt part2.txt part3.txt > combined.txt
cat combined.txt

date | tee timestamp.txt
cat timestamp.txt

echo "first append"  | tee -a timestamp.txt
echo "second append" | tee -a timestamp.txt
cat timestamp.txt
```

**Human-Readable Breakdown:** Create three small files, glue them together with `cat` and `>`, then use `tee` to write a file while still seeing the output on screen. `tee -a` is the append form — the `>>` of the `tee` world.

**Reading it left to right:** `cat part1.txt part2.txt part3.txt` writes the three files' bytes to stdout in argument order; `>` captures that combined stream into `combined.txt`. `date | tee timestamp.txt` runs `date`, pipes its stdout into `tee`, which writes to `timestamp.txt` **and** to its own stdout, which still goes to the terminal.

**The story:** `cat` originally meant **con**catenate. Printing a single file is the degenerate case. When you have log fragments, multipart downloads, or chunked exam evidence, `cat … > combined` is the canonical assembly tool. `tee` exists for the moment you want to watch the output scroll *and* keep a copy — common during installations and long-running jobs.

**Expected output:**

```text
alpha
bravo
charlie
Tue May 26 13:03:01 EDT 2026
Tue May 26 13:03:01 EDT 2026
first append
second append
Tue May 26 13:03:01 EDT 2026
first append
second append
```

**Switches**

| Token | Meaning |
|---|---|
| `cat F1 F2 F3` | Print files in argument order, concatenated |
| `\| tee FILE` | Display on stdout **and** write FILE (overwrite mode) |
| `\| tee -a FILE` | Display on stdout **and** append to FILE |
| `echo "TEXT"` | Print TEXT to stdout |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `cat: file: No such file or directory` | The file name is wrong — `ls` first |
| `tee` shows the output but the file is empty | You are running as a user with no write permission on the target — `sudo tee` |
| Combined file has lines out of order | `cat` writes in the **argument order** you supplied — reorder the arguments |

---

### Task 5 — Safety: `set -o noclobber`, `/dev/null`, and exit codes

**Purpose:** Add the safety habits that distinguish a junior shell user from a senior engineer: `noclobber` to prevent accidental overwrites, `/dev/null` to discard noise, and verifying `$?` after a redirection so you know the command succeeded — not just that the file was created.

```bash
cd /tmp/redir-lab

set -o noclobber
echo "data" > journal.txt        # blocked
echo "data" >| journal.txt       # forced
set +o noclobber

ls /etc > /dev/null
echo "exit was: $?"

ls /no-such-place > /tmp/out.txt
echo "exit was: $?"
cat /tmp/out.txt
```

**Human-Readable Breakdown:** Turn on `noclobber` and watch the shell refuse to overwrite an existing file. Use `>|` to override `noclobber` when you really do mean it. Send `ls` output to `/dev/null` to discard it. Then run a failing command and confirm that the exit code still reflects failure — even though redirection itself never fails.

**Reading it left to right:** `set -o noclobber` flips a shell option; the next `>` against an existing file fails. `>|` is the explicit "I know, do it anyway" override. `set +o noclobber` turns the option back off. `> /dev/null` discards stdout. `$?` is the exit status of the previous command — `0` means success, anything else means failure. Note that the failing `ls` left an empty `/tmp/out.txt` because the shell still opened it with `O_TRUNC`.

**The story:** Every senior engineer has at least one war story about typing `>` when they meant `>>`. `noclobber` is the seatbelt. `/dev/null` is the trash can — when you only care that a command ran, not what it said, send its stdout to `/dev/null`. And `$?` is the truth-teller: a successful redirect does not mean a successful command. Check the exit code or you will ship bugs.

**Expected output:**

```text
bash: journal.txt: cannot overwrite existing file
exit was: 0
ls: cannot access '/no-such-place': No such file or directory
exit was: 2
```

**Switches**

| Token | Meaning |
|---|---|
| `set -o noclobber` | Refuse `>` on existing files in this shell |
| `set +o noclobber` | Turn the option back off |
| `>\|` | Force overwrite even under `noclobber` |
| `> /dev/null` | Discard stdout into the kernel's null device |
| `$?` | Exit status of the previous foreground command |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `>|` does nothing different from `>` | `noclobber` is off — turn it on first with `set -o noclobber` |
| `$?` is always `0` even on failure | You are checking it after `echo`/`cat`/etc — check it **immediately** after the command of interest |
| A failed command still produced a file | `>` truncates the file *before* the command runs; failure does not roll back |

---

### Task 6 — Capstone: RHCSA-realistic stdout capture

**Task statement:** *"Save the output of `ps -ef` to `/root/ps.txt`. Then append a timestamped header line to the same file without losing the existing content. Verify the file contains both the original output and the new header."*

**Purpose:** Execute a full exam-style answer end-to-end, then verify the file the way a grader would.

```bash
sudo -i
cd /root

ps -ef > /root/ps.txt
wc -l /root/ps.txt
head -n 3 /root/ps.txt

echo "# Captured by $(whoami) at $(date -Is) on $(hostname)" >> /root/ps.txt
tail -n 5 /root/ps.txt
wc -l /root/ps.txt

grep -c "^#" /root/ps.txt
test -s /root/ps.txt && echo "VERIFY: file exists and is non-empty"
```

**Human-Readable Breakdown:** Become root, capture the full process listing with `>` (the file may already exist — the prompt says "save," so an overwrite is fine), then append a timestamped header line with `>>` so the original content is preserved. Verify with `head`, `tail`, `wc -l`, and a `grep -c` count.

**Layer stack you built:**

```text
/root/ps.txt                      <- the artifact a grader reads
├── ps -ef output                  <- written with `>` (overwrite-create)
└── # Captured by root at ...      <- appended with `>>` (preserve existing)
```

**The story:** This is the **canonical 90-second exam answer.** Memorize the spine: `cmd > /path/file → cmd >> /path/file → verify with cat/head/tail/wc`. The command and path change with each question, but the order does not. Always verify — never trust that "the command did not error" means "the file is right."

**Expected verification output:**

```text
243 /root/ps.txt
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 12:48 ?        00:00:01 /usr/lib/systemd/systemd ...
root           2       0  0 12:48 ?        00:00:00 [kthreadd]
ec2-user    2099    2098  0 12:51 pts/0    00:00:00 ps -ef
# Captured by root at 2026-05-26T13:05:11-04:00 on ip-10-0-0-12
244 /root/ps.txt
1
VERIFY: file exists and is non-empty
```

**Cleanup**

```bash
rm -rf /tmp/redir-lab
rm -f /root/ps.txt
exit
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `Permission denied` writing to `/root/ps.txt` | You are not root — `sudo -i` first |
| The header line is at the top, not the bottom | You used `>` instead of `>>` for the header — restart and use `>>` |
| `wc -l` shows fewer lines than expected | A command in the chain failed silently — check exit codes |
| `test -s` printed nothing | The file is zero-length — the `ps -ef > ...` step failed |

---

## 🔍 stdout Redirection Decision Guide

```
Got command output you want to capture?
  │
  ├── "Save it to a new file (or replace the old one)"
  │       └── ✅ cmd > file
  │
  ├── "Add it to an existing file without losing the prior content"
  │       └── ✅ cmd >> file
  │
  ├── "Watch it scroll AND save it"
  │       └── ✅ cmd | tee file        (overwrite)
  │       └── ✅ cmd | tee -a file     (append)
  │
  ├── "Discard it — I just want the command to run quietly"
  │       └── ✅ cmd > /dev/null
  │
  ├── "Join several existing files into one"
  │       └── ✅ cat f1 f2 f3 > combined
  │
  ├── "Protect myself against fat-fingered overwrites"
  │       └── ✅ set -o noclobber       (then > refuses to clobber)
  │       └── ✅ >|                     (explicit override when needed)
  │
  └── "I need stderr too"
          └── ✅ See Lab 04: `&>` and `2>&1`
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Set up `/tmp/redir-lab`, inspect FDs, and capture your first line with `>`
- [ ] 02 Use `>` to overwrite a file twice and observe the truncate-on-write semantic
- [ ] 03 Build a journal with `>>` and watch lines accumulate
- [ ] 04 Concatenate files with `cat …  > combined` and split output with `tee` / `tee -a`
- [ ] 05 Practice `set -o noclobber`, `>|`, `/dev/null`, and `$?` checks
- [ ] 06 Execute the RHCSA capstone — `ps -ef > /root/ps.txt` then `>>` a header line

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| `>` instead of `>>` on a log file | Prior log content gone | Always `>>` for accumulating outputs |
| Forgot quotes around a variable in the path | `> $UNDEFINED/file` writes to `/file` | Quote: `> "$DIR/file"` |
| Redirected stdout but command writes errors | File looks empty even though screen showed text | Errors went to stderr — see Labs 02 / 04 |
| Checked `$?` after `cat`, not after the real command | Always sees `0` | Check `$?` **immediately** after the command of interest |
| Used `cmd > file > file2` | Only `file2` gets content | Multiple `>` on one line — only the last one wins; use `tee` instead |
| Edited a file with `>` against an open log | Daemon stops writing or writes to a hole | Use `>>` or `truncate -s 0` instead |
| `cat file \| grep pattern` | Works but wastes a process | `grep pattern file` is cleaner |
| Forgot `-a` on `tee` | Existing file gets truncated | `tee -a file` for append mode |
| Created a file in a directory that does not exist | `No such file or directory` | `mkdir -p` the parent first |
| Wrote to a path you cannot write | `Permission denied` | `sudo -i` or `sudo tee` |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Memorize the spine `cmd > /path/file → verify with cat | head | wc -l`. Every "save the output" task on EX200 reduces to this. Task 6's capstone is intentionally a one-minute drill — be able to do it from a blank shell with no notes.

**RHCE candidate**
- Ansible's `command:` and `shell:` modules return `result.stdout`. Translating *"save the output of `ps -ef` to `/root/ps.txt`"* into Ansible looks like `shell: ps -ef > /root/ps.txt` with `register: ps_result` — and on the controller you have `ps_result.stdout` available without re-reading the file.

**SRE / Platform interview**
- "How do you capture command output during an incident without overwriting prior captures?" → `cmd >> /tmp/incident-$(date +%F).log`, ideally with `set -o noclobber` on for the session.

**DevOps**
- Every "upload build logs as artifacts" CI step is `npm test > test.log` (or equivalent). Knowing `tee` lets you both watch the build live **and** archive the log.

**AI / MLOps**
- `python train.py --lr 3e-4 > runs/lr-3e-4.log` is the universal experiment-capture pattern. Combine with `tee` to watch live training while writing a permanent record.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 02 — Standard Error Redirection (`2>`, `2>/dev/null`) | The sibling: redirect FD 2 instead of FD 1 |
| Lab 03 — Pipe Text Streams (`\|`, `less`, `grep`, `tee`, `wc -l`) | Chain `>` with `\|` to build pipelines |
| Lab 04 — Capture Both Output and Error (`&>`, `2>&1`) | The combined-stream version of this lab |
| Lab 19 — Concatenating Files with `cat` | Deep dive on `cat`, heredocs, and concatenation edge cases |
| Lab 21 — Monitoring Live Log Files (`tail -f`) | Reading the files `>>` keeps growing |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)

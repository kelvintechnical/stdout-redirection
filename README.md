# lab-01a — stdout redirection — RHCSA

Hand-typed RHCSA muscle memory for the canonical stdout redirection
operators: `>`, `>>`, `<`, `|`, and `tee -a`. Built per the rules in
`cursor-adhd-lab-prompt.txt` (sections 0–20). Two tasks, no more.

Task 1 is the canonical correct form. Task 2 is the contrast — the
silent-overwrite trap, `set -o noclobber`, and the `sudo -u` redirection
gotcha that catches everyone exactly once.

---

## LAB HEADER (confirm or correct before Task 1)

```
ENV:   BAREMETAL
DISK:  /dev/sda
NIC:   ens3
SE:    $(getenforce 2>/dev/null || echo n/a)
OS:    $(grep PRETTY_NAME /etc/os-release | cut -d= -f2 | tr -d '"')
TIME:  $(date -Is)
USER:  $(whoami)@$(hostname -s)

TRAPS THIS LAB: T41 T44
PRACTICE DIR:   /tmp — sandbox scratch space; cleared on reboot; safe to write without sudo
```

Run the four `$()` substitutions in your shell to fill in the live values,
then paste the resolved block back so we agree on the environment.

---

## LAB-WIDE SETUP (run once before Task 1; paste output)

```bash
export LAB_NUM=01
export LAB_SLUG=stdout-redirection
export SANDBOX=/tmp/labsandbox_${LAB_NUM}
export GROUP=labgrp_${LAB_NUM}_${LAB_SLUG}
# Never use USER= — bash reserves it; sudo -i resets it to root silently
export LAB_USER=labuser_${LAB_NUM}_${LAB_SLUG}
export LAB_USER_HOME=${SANDBOX}/home_${LAB_USER}

mkdir -p "${SANDBOX}" "${LAB_USER_HOME}"
getent group  "${GROUP}"    >/dev/null || groupadd "${GROUP}"
getent passwd "${LAB_USER}" >/dev/null || useradd \
    -d "${LAB_USER_HOME}" -M -s /bin/bash -g "${GROUP}" "${LAB_USER}"
chown -R "${LAB_USER}:${GROUP}" "${SANDBOX}"
id    "${LAB_USER}"
ls -ld "${SANDBOX}" "${LAB_USER_HOME}"

cat > "${SANDBOX}/THIS_DIRECTORY.txt" <<'EOF'
/tmp is sandbox scratch space; cleared on reboot.
RHCSA labs use it because nothing here survives reboot and no sudo is needed to write.
EOF

echo "Sandbox built by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

---

## TASK 1 of 2 — Canonical stdout redirection

```
LAB:   lab-01a — stdout redirection
TASK:  1 of 2 — > and >> create and append cleanly
TRAPS: T44 (cleanup orphan audit, validated by Task 1 cleanup)
```

### Quiz warm-up (entry baseline — no previous task in this topic)

- **Q1:** After `false`, what does `$?` show?
- **Q2:** What do you think `>` does when you run `echo hi > file.txt`?

Type your answers. I confirm or correct, then we proceed.

---

### Step 1 of 5 — Create a file with `>`

Run this:

```bash
echo "first line" > "${SANDBOX}/notes.txt"
```

Before I explain — what do you think `>` does to `notes.txt` if it
already exists? (Type your guess or "unsure".)

**After you've answered:**

`>` truncates the file to zero bytes, then writes stdout into it. If the
file does not exist, `>` creates it. There is no warning before the
truncation. The full line reads: send the string `first line` (plus a
newline) into `notes.txt` under the sandbox, replacing any prior contents.

Paste your output. You should see no terminal output — the redirection
consumed it.

---

### Step 2 of 5 — Verify with `cat`

Run this:

```bash
cat "${SANDBOX}/notes.txt"
echo "exit was: $?"
```

Before I explain — what should `$?` be after `cat` on a file that exists
and is readable?

**After you've answered:**

`$?` holds the exit status of the previous command. `cat` succeeds → 0.
`cat` on a missing file → non-zero, which is a Section 8 hard blocker.
You should see `first line` and then `exit was: 0`.

Paste your output.

---

### Step 3 of 5 — Append with `>>`

Run this:

```bash
echo "second line" >> "${SANDBOX}/notes.txt"
```

Before I explain — what does the second `>` add? Why two of them?

**After you've answered:**

`>>` is append. Write stdout to the END of the file without truncating.
Two `>` characters together is the explicit append operator. Forgetting
the second one (typing `>` instead of `>>`) silently destroys the file's
prior contents — that is the canonical stdout-redirection trap.

The full line reads: append the string `second line` (plus a newline) to
the existing notes.txt, leaving `first line` intact.

Paste your output.

---

### Step 4 of 5 — Count lines with `wc -l < file`

Run this:

```bash
wc -l < "${SANDBOX}/notes.txt"
```

Before I explain — what does `-l` ask wc to count, and what does `<` do
here?

**After you've answered:**

- `-l` = count lines (newline characters in the input).
- `<` = file-to-stdin redirection. With `<`, `wc` reads the file as if
  it were typed at the terminal, so the output is just the number — no
  filename. Without `<` (i.e. `wc -l file`), you'd see
  `2 /tmp/labsandbox_01/notes.txt`. The `< file` form is the cleanest
  when you want a number you can capture in `$()`.

Paste your output. You should see `2`.

---

### Step 5 of 5 — Append AND display with `tee -a`

Run this:

```bash
echo "third line" | tee -a "${SANDBOX}/notes.txt"
echo "exit was: $?"
```

Before I explain — what does `|` route, and what does `-a` change about
tee?

**After you've answered:**

- `|` = pipe. Connects stdout of the left command to stdin of the right.
- `tee` duplicates stdin to stdout AND a file (named after the plumbing
  T-fitting).
- `tee -a` appends to the file. Without `-a`, tee truncates — same trap
  as `>` versus `>>`.

This line prints `third line` to your terminal AND appends it to
notes.txt. Both happen at once, which is why tee is the standard tool
for "log this AND let me see it scroll by".

Paste your output. You should see `third line` printed and
`exit was: 0`.

---

### Concept card (Task 1)

| Concept | What it does | Exam trap |
|---------|--------------|-----------|
| `>` | truncate-then-write stdout to file | silently destroys prior contents |
| `>>` | append stdout to file | one `>` instead of two = data loss |
| `<` | redirect file to stdin | `wc -l < f` prints just the number |
| `\|` | pipe stdout to next command | only stdout flows; stderr does not (Lab 02 territory) |
| `tee -a` | duplicate stdin to terminal AND file (append) | without `-a`, tee truncates |
| `wc -l` | count newline-terminated lines | unterminated final line is not counted |
| `$?` | exit status of previous command | non-zero = Section 8 blocker |
| `${SANDBOX}` | per-lab safe scratch dir under /tmp | use this, never `$HOME` or bare `/tmp` |

Drill mapping: every row above → `--category io`.

---

### Persistence check

If we rebooted right now, would `${SANDBOX}/notes.txt` survive? What
proves it?

```bash
findmnt /tmp
ls -ld "${SANDBOX}"
```

Paste the output and read the `FSTYPE` column of `findmnt`:

- If FSTYPE is `tmpfs`, /tmp is RAM-backed → `notes.txt` is gone on
  reboot.
- If FSTYPE is something else (xfs / ext4), the file lives on disk but
  `systemd-tmpfiles` will clean /tmp on next boot anyway.

Either way the answer is: it does NOT survive. /tmp is the right home
for sandbox state precisely because of this. Storing real configuration
here is T41 in disguise.

---

### Journal write (run before cleanup)

```bash
LAB=lab01
TASK=task1
JDIR="/root/rhcsa_journal/${LAB}/${TASK}"
mkdir -p "$JDIR"

cat > "$JDIR/done.txt" <<EOF
LAB:    lab-01a-stdout-redirection-rhcsa
TASK:   1 of 2 — > and >> create and append cleanly
DATE:   $(date -Is)
USER:   $(whoami)@$(hostname -s)
STATUS: COMPLETE
EOF

cat > "$JDIR/notes.txt" <<EOF
TOPIC:    stdout redirection — > / >> / | / tee / wc / <
COMMANDS: echo, cat, wc -l, tee -a, findmnt, ls
TRAPS:    T44 (cleanup orphan audit, validated by next block)
MISSED:   [list any quiz question you got wrong, or "none"]
NEXT:     task2 — contrast: silent-overwrite trap + sudo -u LAB_USER + stat
EOF

echo "Journal written: $(ls -la $JDIR)"
echo "exit was: $?"
```

Paste output.

---

### Cleanup (Section 6 teardown — run at end of every task)

```bash
set +e

podman ps -aq --filter "name=^${CTR}$" 2>/dev/null \
    | xargs -r podman rm -f >/dev/null 2>&1

awk -v s="${SANDBOX}" '$2 ~ s {print $2}' /proc/mounts \
    | tac | xargs -r -n1 umount -l 2>/dev/null

if vgs "${VG}" >/dev/null 2>&1; then
    lvremove -fy  "${VG}"          2>/dev/null
    vgremove -fy  "${VG}"          2>/dev/null
fi

losetup -j "${SANDBOX}/disk.img" 2>/dev/null \
    | cut -d: -f1 | xargs -r losetup -d 2>/dev/null

if getent passwd "${LAB_USER}" >/dev/null 2>&1; then
    userdel -r "${LAB_USER}" 2>/dev/null
fi
if getent group "${GROUP}" >/dev/null 2>&1; then
    groupdel "${GROUP}"  2>/dev/null
fi

rm -rf "${SANDBOX}"

echo "── cleanup audit ──"
getent passwd "${LAB_USER}"  && echo "user remains (FAIL)"   || echo "user gone (OK)"
getent group  "${GROUP}"     && echo "group remains (FAIL)"  || echo "group gone (OK)"
test -d "${SANDBOX}"         && echo "sandbox remains (FAIL)" || echo "sandbox gone (OK)"

set -e
echo "Cleanup complete by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

Paste the audit lines. Every row must say `(OK)`. Any `(FAIL)` is a
Section 8 mistake — fix it before continuing.

**STOP.** Task 2 is below. Do not look at it until you have pasted
the five step outputs, the persistence check, the journal write, and
all three audit `(OK)` rows.

---

## TASK 2 of 2 — Contrast: silent-overwrite trap, noclobber, and sudo redirection

This task DELIBERATELY destroys data so you feel why `>` versus `>>`
matters. It also exposes the `sudo -u user cmd > file` gotcha that
makes a file end up root-owned even though you ran the command "as"
the user.

```
LAB:   lab-01a — stdout redirection
TASK:  2 of 2 — silent-overwrite + noclobber + sudo redirection + stat
TRAPS: T41 (persistence reasoning), T44 (cleanup orphan audit)
```

### Quiz warm-up (from Task 1)

- **Q1:** What does `tee -a` do that plain `tee` does not?
- **Q2:** What does `<` do in `wc -l < file`?

Confirm or correct before we proceed.

---

### Prerequisite — re-run the lab-wide setup

The Task 1 cleanup tore down `${LAB_USER}`, `${GROUP}`, and `${SANDBOX}`.
Re-run the **LAB-WIDE SETUP** block at the top of this file before
Step 1. Paste the same `Sandbox built by ...` line as proof.

---

### Step 1 of 5 — Build a multi-line file the right way

Run this:

```bash
echo "alpha"   >  "${SANDBOX}/notes.txt"
echo "bravo"   >> "${SANDBOX}/notes.txt"
echo "charlie" >> "${SANDBOX}/notes.txt"
cat               "${SANDBOX}/notes.txt"
```

Before I explain — three of these lines used `>>`, the first used `>`.
Why does the first one HAVE to be `>` and not `>>`?

**After you've answered:**

The first `>` truncates whatever was there (or creates the file fresh).
If you used `>>` for line 1, you would append `alpha` to whatever the
file already contained — including stale data from a previous run.
`>` first, then `>>` after is the canonical "build a file from scratch"
idiom.

Paste your output. You should see `alpha`, `bravo`, `charlie`.

---

### Step 2 of 5 — The trap: a single `>` destroys the file

Run this. It WILL destroy the file — that is the point.

```bash
echo "newest" > "${SANDBOX}/notes.txt"
cat              "${SANDBOX}/notes.txt"
```

Before I explain — predict what `cat` shows now. (Type your guess.)

**After you've answered:**

You see ONLY `newest`. `alpha`, `bravo`, `charlie` are gone. `>`
truncated the file before writing. There was no warning. There is no
recycle bin. This is the canonical "I meant `>>` and typed `>`"
data-loss event. On a real system this happens to log files, configs,
and `~/.ssh/authorized_keys`.

Paste your output.

---

### Step 3 of 5 — Mitigation: `set -o noclobber` + `>|` to force

Run this:

```bash
set -o noclobber
echo "this should fail" > "${SANDBOX}/notes.txt"
echo "exit was: $?"
```

Before I explain — what do you think `set -o noclobber` does?

**After you've answered:**

`set -o noclobber` tells the shell to refuse `>` against a file that
already exists. The shell returns an error and `$?` is non-zero. The
existing data is preserved. You should see
`bash: ...notes.txt: cannot overwrite existing file` and
`exit was: 1`.

Now force the overwrite (when you actually mean it):

```bash
echo "intentional overwrite" >| "${SANDBOX}/notes.txt"
cat                             "${SANDBOX}/notes.txt"
set +o noclobber
```

`>|` forces `>` even with noclobber on. Use this when you genuinely want
to truncate. `set +o noclobber` turns the protection off again.

Paste your output.

---

### Step 4 of 5 — Write a file as `${LAB_USER}` (the redirection gotcha)

The naive way that DOES NOT do what you think:

```bash
sudo -u "${LAB_USER}" echo "owned by labuser?" > "${SANDBOX}/labuser_note.txt"
stat -c '%U:%G %a %n' "${SANDBOX}/labuser_note.txt"
```

Before I explain — predict who owns `labuser_note.txt`. (Type your
guess.)

**After you've answered:**

The owner is `root`, NOT `${LAB_USER}`. The shell evaluates `>` BEFORE
it runs `sudo`. The redirection happens in your root shell — `>` opens
the file as root, and only THEN does sudo run `echo` as `${LAB_USER}`.
The echo's stdout flows back into the file root already opened.

The right way — let the privileged tool do the writing:

```bash
echo "owned by labuser" | sudo -u "${LAB_USER}" tee "${SANDBOX}/labuser_note.txt" >/dev/null
stat -c '%U:%G %a %n' "${SANDBOX}/labuser_note.txt"
```

Now `tee` runs AS `${LAB_USER}` — it opens the file with `${LAB_USER}`'s
identity, so `${LAB_USER}:${GROUP}` owns the bytes. The `>/dev/null`
discards tee's terminal echo because we don't need to see it scroll by.

Paste both `stat` outputs. The first should show `root:root`, the
second should show `labuser_01_stdout-redirection:labgrp_01_stdout-redirection`.

---

### Step 5 of 5 — Verify with `stat` and a final audit

Run this:

```bash
stat -c '%U:%G %a %n' "${SANDBOX}/labuser_note.txt" \
                      "${SANDBOX}/notes.txt"
ls -lZ "${SANDBOX}/" 2>/dev/null || ls -l "${SANDBOX}/"
echo "exit was: $?"
```

Before I explain — what is `-c '%U:%G %a %n'` formatting?

**After you've answered:**

`stat -c FORMAT` prints only the fields you ask for. `%U` = user owner,
`%G` = group owner, `%a` = octal mode, `%n` = filename. This format
string is the RHCSA-canonical one-liner for auditing a file.

`ls -lZ` adds the SELinux context column. On systems without SELinux
the `-Z` is a no-op so we fall back to plain `ls -l`.

Paste your output.

---

### Concept card (Task 2 contrast — adds to Task 1's card)

| Concept | What it does | Exam trap |
|---------|--------------|-----------|
| `>` then `>>` | canonical "build from scratch" idiom | using `>>` first appends to stale data |
| accidental `>` | silently truncates | data-loss event, no warning, no undo |
| `set -o noclobber` | refuse `>` on existing files | OFF by default — has to be enabled per shell |
| `>\|` | force overwrite under noclobber | only meaningful when noclobber is on |
| `sudo -u USER cmd > file` | redirection runs in the OUTER shell | file ends up root-owned, not USER-owned |
| `\| sudo -u USER tee file` | redirection runs as USER | file ends up USER-owned (correct) |
| `stat -c '%U:%G %a %n'` | one-line audit of ownership and mode | the RHCSA-canonical inspection format |
| `${LAB_USER}` sandbox identity | per-lab user, dies on cleanup | never use bash's `$USER` for this |

Drill mapping: every row above → `--category io`.

---

### Persistence check

If we rebooted right now, would `${SANDBOX}/labuser_note.txt` survive?
What proves it? And does `${LAB_USER}` survive reboot?

```bash
findmnt /tmp
getent passwd "${LAB_USER}"
```

Paste output. Reading:

- `findmnt /tmp` — same as Task 1: tmpfs or filesystem-with-cleanup;
  either way the file is gone on reboot.
- `getent passwd ${LAB_USER}` — the user IS in `/etc/passwd` (a real
  file on disk), so the user account WOULD survive reboot. That is
  exactly why the cleanup block at the end of every task is mandatory:
  T44 = leaving an orphan `${LAB_USER}` on the box, which the next lab
  inherits. The cleanup-audit `(OK)`/`(FAIL)` rows are the only proof.

---

### Journal write (run before cleanup)

```bash
LAB=lab01
TASK=task2
JDIR="/root/rhcsa_journal/${LAB}/${TASK}"
mkdir -p "$JDIR"

cat > "$JDIR/done.txt" <<EOF
LAB:    lab-01a-stdout-redirection-rhcsa
TASK:   2 of 2 — silent-overwrite + noclobber + sudo redirection + stat
DATE:   $(date -Is)
USER:   $(whoami)@$(hostname -s)
STATUS: COMPLETE
EOF

cat > "$JDIR/notes.txt" <<EOF
TOPIC:    stdout redirection traps — silent overwrite, noclobber, sudo redirection
COMMANDS: echo, cat, set -o noclobber, >|, sudo -u, tee, stat, ls -lZ
TRAPS:    T41 (persistence — /tmp volatile but ${LAB_USER} survives)
          T44 (cleanup orphan ${LAB_USER}/${GROUP} audit)
MISSED:   [list any quiz question or step you got wrong, or "none"]
NEXT:     lab-01b-stdout-redirection-trapdrill (Section 18 boundary lab)
EOF

echo "Journal written: $(ls -la $JDIR)"
echo "exit was: $?"
```

Paste output.

---

### Cleanup (Section 6 teardown — final, full audit)

```bash
set +e

podman ps -aq --filter "name=^${CTR}$" 2>/dev/null \
    | xargs -r podman rm -f >/dev/null 2>&1

awk -v s="${SANDBOX}" '$2 ~ s {print $2}' /proc/mounts \
    | tac | xargs -r -n1 umount -l 2>/dev/null

if vgs "${VG}" >/dev/null 2>&1; then
    lvremove -fy  "${VG}"          2>/dev/null
    vgremove -fy  "${VG}"          2>/dev/null
fi

losetup -j "${SANDBOX}/disk.img" 2>/dev/null \
    | cut -d: -f1 | xargs -r losetup -d 2>/dev/null

if getent passwd "${LAB_USER}" >/dev/null 2>&1; then
    userdel -r "${LAB_USER}" 2>/dev/null
fi
if getent group "${GROUP}" >/dev/null 2>&1; then
    groupdel "${GROUP}"  2>/dev/null
fi

rm -rf "${SANDBOX}"

echo "── cleanup audit ──"
getent passwd "${LAB_USER}"  && echo "user remains (FAIL)"   || echo "user gone (OK)"
getent group  "${GROUP}"     && echo "group remains (FAIL)"  || echo "group gone (OK)"
test -d "${SANDBOX}"         && echo "sandbox remains (FAIL)" || echo "sandbox gone (OK)"

set -e
echo "Cleanup complete by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

Paste the audit lines. Every row must say `(OK)`.

---

### Drill (run AFTER cleanup audit shows all OK)

```bash
python3 ~/scripts/rhcsa_drill.py --category io
```

Paste your score. If <80%, drill `--category io` again before starting
lab-01b. Per Section 20, this score gate is mandatory.

---

### Rotation tracker update (last step of the lab)

```bash
echo "last_used=01" > /root/rhcsa_journal/dir_rotation.txt
cat                  /root/rhcsa_journal/dir_rotation.txt
```

Next lab's practice directory will be `/etc` (rotation slot 02).

**STOP — lab-01a complete.** Begin lab-01b only after the drill score is
pasted. lab-01b is a TRAP DRILL LAB per Section 18 of the prompt:
Task 1 is a wrong-way demo of the primary stdout-redirection trap,
Task 2 is the `ansible.builtin.shell:` boundary statement.

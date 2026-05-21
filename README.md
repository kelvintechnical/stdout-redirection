# Lab: stdout Redirection — `>`, `>>`, and Beyond

**RHCSA EX200 Skill** | Series: File Operations & Shell Fundamentals  
**Prerequisite:** Basic Linux shell familiarity  
**Time Estimate:** ~20 minutes  
**Covers RHCSA Tasks:** 11 (archive output), 14 (find output to file), 19 (regex output to file)

---

## 🎯 Objective

Master Linux I/O redirection — how to control where command output goes. This is not just a shell trick: every RHCSA task that says *"save the output to a file"* or *"capture the result"* requires this skill.

---

## 🧠 Concept: How Linux Handles Output

Every process in Linux has three standard data streams open by default:

| Stream | Number | Name | Default destination |
|---|---|---|---|
| **stdin** | 0 | Standard Input | Keyboard |
| **stdout** | 1 | Standard Output | Terminal screen |
| **stderr** | 2 | Standard Error | Terminal screen |

When you run `ls`, the file list goes to **stdout** — your screen. Redirection lets you point that stream somewhere else, like a file.

```
Normal:   command → stdout → screen
Redirect: command → stdout → file
```

> **Why this matters on the RHCSA exam:** Tasks 14, 18, and 19 in the sample exam explicitly say "save output to a file." Without redirection, you cannot complete them.

---

## 🗺️ Path Reference: `./` vs `~/` vs `cd`

Think of it like a **physical location**:

| You are at | Analogy |
|---|---|
| Your terminal prompt | You standing somewhere in a building |
| `cd` | Walking to a different room |
| `~/` | Your apartment number — a fixed address |
| `./` | "Right here where I'm standing" |

---

### `./` — Right here

```bash
# You're in /home/ec2-user/redirection-lab
./script.sh        # Run script.sh IN THIS FOLDER
```

Without `./`, Linux won't look in the current folder for programs — it only searches folders in your `$PATH`. `./` forces it to look right where you are.

---

### `~/` — Your home address

```bash
~/redirection-lab       # = /home/ec2-user/redirection-lab
~/documents/file.txt    # = /home/ec2-user/documents/file.txt
```

`~/` is just a **shortcut** — it always expands to your home directory regardless of where you currently are. You're not moving anywhere — you're just referencing a full path.

---

### `cd` — Actually move

```bash
cd ~/redirection-lab    # Now you ARE in /home/ec2-user/redirection-lab
cd /var/tmp             # Now you ARE in /var/tmp
cd ~                    # Go back home
cd ..                   # Go up one directory
```

---

### The key difference

```bash
# These do the SAME thing two different ways:

# Option 1 — reference with ~/
rm -rf ~/redirection-lab    # Delete it from wherever you are

# Option 2 — move first, then act
cd ~
rm -rf redirection-lab      # Now you're home, so no ~/ needed
```

> **Rule of thumb:** Use `~/` when you want to reference a path without moving. Use `cd` when you need to work inside a directory for multiple commands.

---

## 📚 Operator Reference

| Operator | Behavior | File exists? | File missing? |
|---|---|---|---|
| `>` | Write stdout to file | **Overwrites** existing content | Creates new file |
| `>>` | Append stdout to file | **Adds to end** of existing content | Creates new file |
| `2>` | Write stderr to file | Overwrites | Creates new file |
| `2>>` | Append stderr to file | Adds to end | Creates new file |
| `&>` | Write both stdout and stderr to file | Overwrites | Creates new file |
| `2>&1` | Redirect stderr into stdout stream | Combined output | Combined output |
| `tee` | Write to file AND display on screen | Overwrites (use `-a` to append) | Creates new file |

> **Most common mistake:** Using `>` when you meant `>>` — silently destroys the existing file contents. There is no undo.

---

## 🔧 Steps

### Step 1 — Create your working directory

```bash
mkdir -p ~/redirection-lab
cd ~/redirection-lab
```

**Output:** *(none — silence means success)*

> `mkdir -p` creates the directory and any missing parent directories in one command. The `-p` flag prevents an error if the directory already exists.

---

### Step 2 — Write output to a new file with `>`

```bash
echo "First line" > output.txt
cat output.txt
```

**Expected output:**

```
First line
```

| Part | Meaning |
|---|---|
| `echo "First line"` | Prints the string to stdout |
| `>` | Redirects stdout away from the screen |
| `output.txt` | Destination file — created if it doesn't exist |

> `cat` (concatenate) reads a file and prints its contents to stdout. It is the standard way to verify file contents on the command line.

---

### Step 3 — Overwrite behavior of `>`

```bash
echo "Replacement line" > output.txt
cat output.txt
```

**Expected output:**

```
Replacement line
```

"First line" is **gone**. `>` truncated the file to zero bytes before writing. It did not ask for confirmation. This is permanent — there is no recycle bin in Linux.

> ⚠️ **`>` is destructive.** On the RHCSA exam, if a task says "save output to a file" and that file already has content you need, always use `>>` to be safe.

---

### Step 4 — Append output with `>>`

```bash
echo "Line 1" > output.txt
echo "Line 2" >> output.txt
echo "Line 3" >> output.txt
cat output.txt
```

**Expected output:**

```
Line 1
Line 2
Line 3
```

| Command | File state after |
|---|---|
| `echo "Line 1" > output.txt` | File contains: `Line 1` |
| `echo "Line 2" >> output.txt` | File contains: `Line 1`, `Line 2` |
| `echo "Line 3" >> output.txt` | File contains: `Line 1`, `Line 2`, `Line 3` |

`>>` never touches existing content — it moves to the end of the file and adds from there.

---

### Step 5 — Redirect real command output

```bash
ls /etc > etc-listing.txt
cat etc-listing.txt | head -5
```

**Expected output (first 5 lines):**

```
DIR_COLORS
DIR_COLORS.256color
DIR_COLORS.lightbgcolor
GREP_COLORS
NetworkManager
```

> `head -5` prints only the first 5 lines — useful when a file has hundreds of entries and you just want to confirm it worked.

Now append the `/var` listing to the same file:

```bash
ls /var >> etc-listing.txt
wc -l etc-listing.txt
```

> `wc -l` counts lines in a file. Confirms both directory listings were captured without either overwriting the other.

---

### Step 6 — Understand stderr (`2>`)

```bash
ls /nonexistent > errors.txt
cat errors.txt
```

The error appeared on screen even though you redirected to a file — and `errors.txt` is empty. Why?

`>` only redirects **stdout (stream 1)**. Error messages go to **stderr (stream 2)** — a completely separate stream.

**To capture stderr:**

```bash
ls /nonexistent 2> errors.txt
cat errors.txt
```

**Expected output:**

```
ls: cannot access '/nonexistent': No such file or directory
```

---

### Step 7 — Capture both stdout and stderr together

```bash
find /etc -name "*.conf" > find-results.txt 2>&1
wc -l find-results.txt
```

| Part | Meaning |
|---|---|
| `> find-results.txt` | Redirect stdout to the file |
| `2>&1` | Redirect stderr (2) into wherever stdout (1) is going — the file |

**Read `2>&1` as:** "Send stream 2 to wherever stream 1 is currently going."

> ⚠️ **Order matters:** `2>&1 > file` is wrong. Always write `> file 2>&1`.

**Shorthand alternative:**

```bash
find /etc -name "*.conf" &> find-results.txt
```

---

### Step 8 — `tee`: write to file AND see output on screen

```bash
ls /etc | tee screen-and-file.txt | head -5
wc -l screen-and-file.txt
```

`tee` sends output to both the screen and the file simultaneously. `head -5` only limits what you see on screen — the file receives everything.

**Append with `tee -a`:**

```bash
echo "appended line" | tee -a screen-and-file.txt
```

---

### Step 9 — RHCSA exam pattern: find + redirect

Task 14: *"Search for all files modified in the past 30 days and save the listing to `/var/tmp/modfiles.txt`."*

```bash
find / -mtime -30 > /var/tmp/modfiles.txt 2>&1
wc -l /var/tmp/modfiles.txt
head -10 /var/tmp/modfiles.txt
```

| Part | Meaning |
|---|---|
| `find /` | Search from filesystem root |
| `-mtime -30` | Modified within the last 30 days (`-` means "less than") |
| `> /var/tmp/modfiles.txt` | Write results to this file |
| `2>&1` | Include permission errors in the file instead of cluttering the screen |

---

### Step 10 — RHCSA exam pattern: grep + redirect

Task 19: *"Lock user70. Use regex to capture the lock line and store it in `/var/tmp/user70.lock`."*

> ⚠️ Create the user first if they don't exist:
> ```bash
> sudo useradd user70
> ```

```bash
sudo usermod -L user70
sudo grep "^user70:!" /etc/shadow > /var/tmp/user70.lock
cat /var/tmp/user70.lock
```

**Expected output:**

```
user70:!:20594:0:99999:7:::
```

**Command breakdown:**

| Part | Meaning |
|---|---|
| `sudo` | Run as root — `/etc/shadow` is only readable by root |
| `grep` | Search for a pattern in a file |
| `^` | Regex — line must **start with** this pattern |
| `user70:!` | Match lines starting with `user70:!` — `!` means account is locked |
| `/etc/shadow` | File storing hashed passwords for all users |
| `>` | Redirect stdout to a file (overwrite) |
| `/var/tmp/user70.lock` | Destination file — created if it doesn't exist |

**`/etc/shadow` field breakdown:**

```
user70 : !      : 20594       : 0        : 99999    : 7        : : : :
user     locked   last change   min age    max age    warning
```

| Field | Value | Meaning |
|---|---|---|
| `user70` | username | The account |
| `!` | locked | `!` prefix = account is locked, can't login |
| `20594` | last change | Days since Jan 1 1970 password was last changed |
| `0` | min age | Minimum days before password can be changed |
| `99999` | max age | Days until password expires (99999 = never) |
| `7` | warning | Days before expiry to warn user |
| `:::` | empty fields | Inactive period, expiry date, reserved — all unset |

---

### Step 11 — Clean up

```bash
cd ~
rm -rf ~/redirection-lab
rm -f /var/tmp/modfiles.txt /var/tmp/user70.lock
```

**Command breakdown:**

| Command | Meaning |
|---|---|
| `cd ~` | Return to home directory before deleting |
| `rm` | Remove |
| `-r` | Recursive — delete directory and everything inside it |
| `-f` | Force — no confirmation prompts, ignore missing files |
| `~/redirection-lab` | Works from **anywhere** — `~/` is an absolute reference to home |
| `rm -f` with two paths | Deletes both files in one command, no prompts |
| `/var/tmp/` | Temporary files directory — survives reboots unlike `/tmp` |

> `~/redirection-lab` works from anywhere because `~/` is an absolute reference to your home directory — no need to `cd` first.

---

## 🔍 Operator Decision Guide

```
Do you want to see output AND save it?
  └── YES → tee (or tee -a to append)
  └── NO  →
        Do you want to keep existing file content?
          └── YES → >> (append)
          └── NO  → > (overwrite)
        
        Do you need errors too?
          └── YES → > file 2>&1  or  &> file
          └── NO  → > file  (stdout only)
```

---

## ✅ Lab Checklist

- [ ] `>` creates and overwrites a file ✅
- [ ] `>>` appends without touching existing content ✅
- [ ] `>` does not capture stderr — confirmed with `ls /nonexistent` ✅
- [ ] `2>` captures stderr separately ✅
- [ ] `2>&1` combines both streams into one file ✅
- [ ] `tee` writes to file and displays on screen simultaneously ✅
- [ ] `find / -mtime -30 > file 2>&1` pattern practiced ✅
- [ ] `grep` output redirected to file using exam-style task ✅

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Using `>` instead of `>>` | Existing file content silently destroyed | Always check — use `>>` if the file has data you need |
| `2>&1 > file` wrong order | stderr goes to screen, not file | Always write `> file 2>&1` — file redirect first |
| Missing `2>&1` on `find /` | Terminal floods with `Permission denied` | Add `2>&1` to send errors to the file |
| Forgetting `sudo` when redirecting to `/var/tmp` | `Permission denied` on the file write | Use `sudo tee` instead: `command | sudo tee /var/tmp/file` |
| `cat` on a huge file | Terminal scrolls uncontrollably | Use `head -20` or `less` instead |
| User doesn't exist before `usermod -L` | `usermod: user 'user70' does not exist` | Run `sudo useradd user70` first |

---

## 📌 RHCSA Exam Strategy

- Any task that says **"save output to a file"** = redirection. Know which operator to use before the exam.
- **`find / ... > file 2>&1`** is the exact pattern for Tasks 13 and 14 — memorize it.
- **`grep "pattern" /file > /var/tmp/outfile`** is the pattern for Task 19.
- **`command >> /var/tmp/file`** when appending to an existing output file — safer than `>`.
- `tee` is useful when exam instructions say to both display and save — uncommon but possible.
- Always **verify with `cat` or `wc -l`** after redirecting — never assume the file was written correctly.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Task 11 — `tar` + `gzip` | Output of tar operations redirected with `>` |
| Task 13 — `cron` + `find` | `find` results saved with `>` in cron job |
| Task 14 — `find` modified files | Direct application of `find / > file 2>&1` |
| Task 18 — `dnf group install` | Capture install output with `tee` or `>` |
| Task 19 — Lock user + grep | `grep` output saved with `>` |

---

## 👤 Author

**Kelvin R. Tobias**  
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)

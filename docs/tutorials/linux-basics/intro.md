---
id: linux-intro
title: Introduction to Linux for HPC Users
sidebar_label: Introduction to Linux
sidebar_position: 1
---

# Introduction to Linux for HPC Users

This tutorial is for researchers who are comfortable using a personal computer — Windows or macOS — but are new to working on the ACE HPC cluster. On a personal computer you click icons, drag windows, and navigate folders visually. On the cluster, you type commands. That shift can feel abrupt. This guide walks you through it step by step.

By the end you will know how to navigate the cluster's filesystem, create and manage files and directories, and write a simple Bash script to automate a repetitive task.

:::info Prerequisites
You should already have an ACE HPC account and be able to log in via SSH. If you haven't done that yet, see [Connecting to the Cluster](../../getting-started/connecting).
:::

---

## 1. Why the Cluster Runs Linux

Your laptop has a graphical desktop where you interact with windows and menus. The ACE HPC cluster runs **Linux** — a free, open-source operating system that powers the vast majority of the world's supercomputers, servers, and cloud infrastructure.

Linux is preferred for HPC for several reasons:

- **Stability and performance** — Linux can run for years without rebooting and handles many simultaneous users gracefully.
- **The command line** — a text-based interface where you describe exactly what you want the computer to do. It is faster, scriptable, and reproducible in ways that point-and-click interfaces are not.
- **Software ecosystem** — most scientific software, compilers, and tools are built for Linux first.


When you connect to the cluster over SSH, you land in a **terminal** — a text window where you type commands and read their output. There is no desktop wallpaper, no file explorer, no taskbar. The terminal *is* the interface.

---

## 2. The Terminal Prompt

After logging in, you will see something like this:

```
katukunda@wireless-10-155-153-46 ~ % ssh akatukunda@biocompace.ace.ac.ug
akatukunda@biocompace.ace.ac.ug's password:
**************************************************************************
*                    ACE OpenHPC 3.2 Software Stack                      *
*                                                                        *
*  Hello HPC User,                                                       *
*                                                                        *
*  Welcome to the ACE-Uganda HPC! For any additional required packages   *
*  and application configuration problems, do not hesitate to reach out  *
*  at:                                                                   *
*  support@ace-bioinformatics.org                                        *
*                                                                        *
*  Visit our wiki: https://ace-bioinformatics.github.io/ace-ug-hpc-wiki/ *
*  Thank you.                                                            *
*                           ACE-Uganda Project                           *
**************************************************************************
Last login: Fri Feb 20 20:43:13 2026 from 10.35.50.27
[akatukunda@kla-ac-hpc-02 ~]$
```

This is the **shell prompt**. It tells you:

| Part | Meaning |
|------|---------|
| `akatukunda` | Your username |
| `kla-ac-hpc-02` | The name of the login node you are on |
| `~` | Your current location (the `~` symbol is shorthand for your home directory) |
| `$` | You are a regular user (not the system administrator) |

Everything you type appears after the `$`. When you press **Enter**, the shell reads your command and runs it.

### Your First Command

Try this now — type it and press Enter:

```bash
whoami
```

The shell prints your username and returns you to the prompt. Every command follows this pattern: you type, you press Enter, the output appears, the prompt returns.

---

## 3. The Linux Filesystem

On Windows, files live on drives like `C:\` or `D:\`. On Linux, everything lives in a single unified tree that starts at the **root**, written as `/`.

```
/
├── home/
│   └── akatukunda/        ← your home directory
├── scratch/               ← fast temporary storage on ACE
├── bin/                   ← system programs
├── usr/
│   └── local/
│       └── software/      ← installed software on ACE
└── tmp/                   ← temporary files
```

### The Directories That Matter on ACE HPC

When you log in, you have access to two main storage areas:

| Location | Variable | Purpose | Notes |
|----------|----------|---------|-------|
| `/home/username` | `$HOME` | Your home directory | Limited quota — for scripts, configs, small files |
| `/scratch/username` | `$SCRATCH` | High-speed scratch space | For large datasets and active job output |

:::warning Keep $HOME lean
Your home directory has a storage quota. Store large datasets and job input/output files in `/scratch`, not in `$HOME`. See [File System Practices](../../good-conduct/file-system) for the full guidelines.
:::

---

## 4. Navigating the Filesystem

### Where am I?

```bash
pwd
```

`pwd` stands for **print working directory**. It tells you exactly where you are in the filesystem right now. You will see something like:

```
/home/akatukunda
```

### What is in this directory?

```bash
ls
```

`ls` **lists** the contents of your current directory. Try some useful variations:

```bash
ls -l        # long format: shows permissions, size, and date
ls -lh       # same, but sizes in human-readable form (KB, MB, GB)
ls -a        # show hidden files (files starting with a dot)
ls -lh /scratch/akatukunda   # list a specific directory without going there
```

### Moving around

```bash
cd /scratch/akatukunda    # go to scratch space
cd ~                       # go back to your home directory (~ always means $HOME)
cd ..                      # go up one level (to the parent directory)
cd -                       # go back to the previous directory
```

Practice: from your home directory, navigate to `/scratch`, look around, then return home.

```bash
pwd                        # confirm you're in $HOME
cd /scratch/akatukunda
ls -lh
cd ~
pwd                        # you're back in $HOME
```

### Paths: absolute vs. relative

An **absolute path** starts from root `/` and always works regardless of where you are:
```
/scratch/akatukunda/project1/data/reads.fastq
```

A **relative path** starts from your current location:
```
project1/data/reads.fastq    # only works if you're inside /scratch/akatukunda
```

The shorthand `.` means "the current directory" and `..` means "one level up":

```bash
ls .       # list the current directory (same as just "ls")
ls ..      # list the parent directory
```

---

## 5. Creating and Managing Files and Directories

### Creating directories

```bash
mkdir projects                        # create a folder called "projects" here
mkdir -p projects/run1/output         # create nested folders in one step
```

The `-p` flag tells `mkdir` to create any missing parent directories — so you don't need to create `projects/` before `projects/run1/` before `projects/run1/output/`.

### Creating files

```bash
touch notes.txt       # create an empty file (or update the timestamp if it exists)
```

### Copying files and directories

```bash
cp notes.txt notes_backup.txt             # copy a file
cp -r projects/ projects_backup/          # copy a directory (-r means recursive)
```

### Moving and renaming

```bash
mv notes.txt archive/notes.txt            # move a file into a directory
mv old_name.txt new_name.txt              # rename a file (same command)
```

### Deleting files and directories

```bash
rm notes.txt                              # delete a file
rm -r projects_backup/                    # delete a directory and everything inside it
```

:::danger rm is permanent
Linux has no recycle bin. When you `rm` a file, it is gone immediately — there is no undo. Double-check what you are deleting before pressing Enter.
:::

### Practical exercise: build a project structure

Create a clean working directory in your scratch space for a new analysis:

```bash
mkdir -p /scratch/$USER/my_project/{data,scripts,results,logs}
ls /scratch/$USER/my_project/
```

The `{data,scripts,results,logs}` notation is **brace expansion** — Bash expands it into four separate `mkdir` calls. You will see all four directories created at once.

---

## 6. Viewing and Searching File Contents

You rarely edit files directly in the terminal at first. More often you want to look at them.

### Displaying file contents

```bash
cat results.txt           # print the entire file to the screen
head -20 results.txt      # show the first 20 lines
tail -20 results.txt      # show the last 20 lines
tail -f job.log           # continuously follow a file as it grows (useful for job logs)
```

### Scrolling through a large file

```bash
less results.txt
```

`less` opens a scrollable viewer. Use the arrow keys or Page Up/Down to scroll. Press `q` to quit.

### Counting lines and words

```bash
wc -l results.txt         # count lines
wc -w results.txt         # count words
```

### Searching inside files

```bash
grep "ERROR" job.log                 # find lines containing "ERROR"
grep -i "warning" job.log            # case-insensitive search
grep -r "sample_001" /scratch/$USER/ # search recursively through a directory
```

---

## 7. File Permissions

Every file and directory on Linux has permissions that control who can read, write, or execute it. When you run `ls -l`, the first column shows this:

```
-rw-r--r-- 1 akatukunda users  4.2K Jan 15 10:23 notes.txt
drwxr-xr-x 3 akatukunda users  4.0K Jan 15 10:30 projects/
```

The permission string breaks down as:

```
- rw- r-- r--
│  │   │   └── others: read only
│  │   └────── group: read only
│  └────────── owner: read and write
└───────────── file type (- = file, d = directory)
```

Each group of three letters is: **r** (read), **w** (write), **x** (execute). A `-` means that permission is not granted.

### Making a script executable

Before you can run a Bash script you have written, you need to give it execute permission:

```bash
chmod +x myscript.sh      # add execute permission for everyone
./myscript.sh              # run it
```

---

## 8. Environment Variables

The shell uses **environment variables** to store configuration values. They are written in UPPERCASE by convention. You access them with a `$` prefix.

```bash
echo $HOME       # your home directory
echo $USER       # your username
echo $PATH       # the list of directories the shell searches for programs
echo $SCRATCH    # your scratch directory on ACE
```

You can set your own:

```bash
export PROJECT="/scratch/$USER/my_project"
echo $PROJECT
ls $PROJECT
```

Variables set this way exist only for the current session. To make them permanent, add the `export` line to your `~/.bashrc` file.

---

## 9. Bash Scripting: A Practical Example

A **Bash script** is a plain text file containing a sequence of shell commands. Instead of typing the same commands every time, you write them once and run the script. This is essential for HPC work, where you need reproducible, automated workflows.

### The practical problem

Suppose you run a bioinformatics pipeline and produce a separate log file for each sample. At the end of a run, you have dozens of log files scattered in your output directory. You want to:

1. Create a summary report listing each sample and whether it completed successfully or failed.
2. Move failed logs to a separate folder for review.
3. Record how many samples passed and how many failed.

This is the kind of task you would quickly automate with a Bash script.

### Writing the script

In your work space, create a directory to follow along:

```bash
mkdir -p bash_tutorial/logs
cd bash_tutorial
```

Generate some dummy log files to work with:

```bash
for sample in sample_001 sample_002 sample_003 sample_004 sample_005; do
    echo "Processing $sample..." > logs/${sample}.log
done

# Simulate two failures
echo "ERROR: alignment failed" >> logs/sample_003.log
echo "ERROR: out of memory"    >> logs/sample_005.log
```

Now create the script. Open a new file called `summarize_run.sh`:

```bash
touch summarize_run.sh
```

Write the following into it using `nano` (a simple terminal text editor):

```bash
nano summarize_run.sh
```

Type or paste the script below. When done, press `Ctrl+O` to save and `Ctrl+X` to exit.

```bash
#!/usr/bin/env bash
# summarize_run.sh
# Checks each sample log for errors and produces a summary report.

# --- Configuration ---
LOG_DIR="logs"
FAILED_DIR="failed_logs"
REPORT="run_summary.txt"

# --- Setup ---
mkdir -p "$FAILED_DIR"
> "$REPORT"   # empty the report file if it already exists

passed=0
failed=0

echo "============================" >> "$REPORT"
echo "  Run Summary Report"        >> "$REPORT"
echo "  $(date)"                   >> "$REPORT"
echo "============================" >> "$REPORT"
echo ""                            >> "$REPORT"

# --- Process each log file ---
for logfile in "$LOG_DIR"/*.log; do
    # Extract the sample name from the filename (strip directory and .log extension)
    sample=$(basename "$logfile" .log)

    if grep -q "ERROR" "$logfile"; then
        echo "FAILED  $sample" >> "$REPORT"
        mv "$logfile" "$FAILED_DIR/"
        ((failed++))
    else
        echo "PASSED  $sample" >> "$REPORT"
        ((passed++))
    fi
done

# --- Print totals ---
echo ""                                      >> "$REPORT"
echo "----------------------------"          >> "$REPORT"
echo "Total passed: $passed"                 >> "$REPORT"
echo "Total failed: $failed"                 >> "$REPORT"
echo "Total samples: $((passed + failed))"   >> "$REPORT"

# Also print the report to the screen
cat "$REPORT"
```

### Understanding the script

Let's walk through the key parts:

**`#!/usr/bin/env bash`** — The **shebang line**. This must be the very first line. It tells the operating system to use Bash to interpret the script.

**Variables** — `LOG_DIR="logs"` sets a variable. Use `"$LOG_DIR"` (with quotes) when you reference it, to handle filenames with spaces safely.

**`> "$REPORT"`** — The `>` operator redirects output *into* a file, overwriting it. Here it is used with no command on the left, which creates an empty file (or empties an existing one). Later, `>>` is used to *append* to the file rather than overwrite it.

**`for logfile in "$LOG_DIR"/*.log`** — A `for` loop that iterates over every `.log` file in the logs directory. The `*.log` pattern is expanded by Bash into the list of matching filenames.

**`if grep -q "ERROR" "$logfile"`** — An `if` statement that checks whether the word "ERROR" appears in the log. `grep -q` runs silently (quiet mode) and returns an exit code: 0 if found, 1 if not. Bash treats exit code 0 as true.

**`basename "$logfile" .log`** — Strips the directory path and the `.log` suffix, leaving just the sample name.

**`((failed++))`** — Arithmetic in Bash uses double parentheses. This increments the `failed` counter by 1.

### Making the script executable and running it

```bash
chmod +x summarize_run.sh
./summarize_run.sh
```

You should see:

```
============================
  Run Summary Report
  Thu Jan 15 14:32:01 UTC 2026
============================

PASSED  sample_001
PASSED  sample_002
FAILED  sample_003
PASSED  sample_004
FAILED  sample_005

----------------------------
Total passed: 3
Total failed: 2
Total samples: 5
```

Verify the failed logs were moved:

```bash
ls failed_logs/
```

```
sample_003.log  sample_005.log
```

This is the core pattern of HPC scripting: loop over inputs, apply a check or transformation, collect results, report. The same structure scales to hundreds of samples.

---

## 10. Helpful Terminal Habits

### Tab completion

Press **Tab** to autocomplete command names, file paths, and directory names. If there are multiple matches, press Tab twice to see them listed. This saves typing and prevents typos.

```bash
cd /scratch/aka<Tab>     # completes to your username
ls my_pro<Tab>           # completes to my_project/ if it exists
```

### Command history

```bash
history              # show all previous commands with numbers
!42                  # re-run command number 42 from history
!!                   # re-run the last command
```

Press the **Up arrow** to cycle through previous commands. Press **Ctrl+R** and start typing to search your history interactively.

### Stopping a command

Press **Ctrl+C** to interrupt and stop a running command immediately.

### Getting help

```bash
man ls              # open the manual page for the ls command
ls --help           # most commands accept --help for a quick summary
```

---

## Quick Reference

| Task | Command |
|------|---------|
| Show current directory | `pwd` |
| List files | `ls -lh` |
| Go to home directory | `cd ~` |
| Go up one level | `cd ..` |
| Create a directory | `mkdir -p dirname` |
| Create a file | `touch filename` |
| Copy a file | `cp source dest` |
| Move or rename | `mv source dest` |
| Delete a file | `rm filename` |
| Delete a directory | `rm -r dirname` |
| View a file | `less filename` |
| Search in a file | `grep "pattern" filename` |
| Count lines | `wc -l filename` |
| Make script executable | `chmod +x script.sh` |
| Run a script | `./script.sh` |
| Show environment variable | `echo $VARIABLE` |

---

## What's Next?

Now that you can navigate the cluster and write basic scripts, the natural next steps are:

- **[SLURM Basics](../running-jobs/slurm-basics)** — learn how to submit computational jobs to the ACE cluster's job scheduler rather than running them on the login node.
- **[Writing Job Scripts](../running-jobs/job-scripts)** — combine your Bash scripting knowledge with SLURM directives to run real workloads on compute nodes.

---

## References

- [GNU Bash Reference Manual](https://www.gnu.org/software/bash/manual/)
- [The Linux Command Line (free book)](https://linuxcommand.org/tlcl.php) by William Shotts
- [Bash scripting cheatsheet](https://devhints.io/bash)

---
id: faq
title: Frequently Asked Questions
---

# Frequently Asked Questions

## Accounts and Access

**Q: How do I reset my password?**
A: Email [support@ace-bioinformatics.org](mailto:support@ace-bioinformatics.org) with your username and request a password reset. You will receive a temporary password with instructions to set a new one.

**Q: I can't SSH into the cluster. What should I check?**
A: Work through these steps in order:

1. Verify you're using the correct hostname and username.
2. Check that you're on a network that can reach the cluster (some institutions require VPN).
3. Make sure your account is active — new accounts require completing the Linux/SLURM quiz.
4. Try `ssh -v user@hostname` for verbose output that reveals where the connection fails.
5. If you still can't connect, email support with the verbose SSH output.

**Q: Can I increase my storage quota?**
A: Yes. Email [support@ace-bioinformatics.org](mailto:support@ace-bioinformatics.org) with your username, current usage, how much additional space you need, and a brief justification. Check your current usage with `quota -s` or `df -h ~`.

---

## Jobs and Scheduling

**Q: Why is my job stuck in the pending state?**
A: Common reasons include:

- **Resources unavailable** — all nodes matching your request are in use. Try reducing `--nodes`, `--ntasks`, `--mem`, or `--time`.
- **Priority** — other jobs submitted earlier or by higher-priority accounts are ahead in the queue.
- **Partition limits** — you may have exceeded the maximum number of concurrent jobs or total resource allocation for your partition.

Check the reason with:

```bash
squeue -u $USER -o "%.18i %.9P %.8j %.8u %.2t %.10M %.6D %R"
```

The last column (`NODELIST(REASON)`) tells you why the job is waiting (e.g., `Priority`, `Resources`, `QOSMaxJobsPerUserLimit`).

**Q: My job was killed unexpectedly. How do I find out why?**
A: Start with these commands:

```bash
# Check job exit status
sacct -j <jobid> --format=JobID,JobName,State,ExitCode,MaxRSS,Elapsed,Timelimit

# Review job output files
cat slurm-<jobid>.out
cat slurm-<jobid>.err
```

Common causes:
- **OOM (Out of Memory)** — the job exceeded its memory allocation. Look for `oom_kill` in the output. Increase `--mem`.
- **Timeout** — the job exceeded its `--time` limit. The `State` will show `TIMEOUT`. Request more time or optimize your workflow.
- **Node failure** — rare, but hardware issues can kill jobs. The `State` will show `NODE_FAIL`. Resubmit the job.

**Q: How do I request the right amount of memory for my job?**
A: If you're unsure, run a test job with a generous memory request and a small dataset. Then check actual usage:

```bash
sacct -j <jobid> --format=JobID,MaxRSS,ReqMem
```

`MaxRSS` shows peak memory used. Request about 20% more than that for your production run to allow for variability across datasets.

**Q: What is the difference between `sbatch`, `srun`, and `salloc`?**
A:

| Command | Purpose | Use when... |
|---------|---------|-------------|
| `sbatch` | Submit a batch job script to the queue | You want the job to run unattended |
| `srun` | Launch a job step (often inside a batch script) | You need to run a parallel task within an allocation |
| `salloc` | Request an interactive allocation | You need to test commands, debug, or run interactive tools |

**Q: Should I run anything on the login node?**
A: Login nodes are shared by all users and should only be used for:

- Editing files and scripts
- Submitting and monitoring jobs (`sbatch`, `squeue`, `scancel`)
- Small file operations and transfers
- Compiling code (short compilations only)

**Never** run compute-intensive tasks, large data processing, or long-running programs on the login node. Use `sbatch` for batch work or `salloc` for interactive computing.

**Q: How do I run an interactive session?**
A: Use `salloc` to request resources, then run commands on the allocated compute node:

```bash
salloc --ntasks=1 --mem=4G --time=02:00:00
# Once allocated, you'll be on a compute node
# Run your commands here
# Type 'exit' when done to release resources
```

This is useful for testing scripts, debugging, and running tools that need interactive input.

**Q: How do I cancel a job?**
A: Use `scancel`:

```bash
scancel <jobid>            # Cancel a specific job
scancel -u $USER           # Cancel all your jobs
scancel -n <job_name>      # Cancel jobs by name
```

**Q: How do I use multiple CPUs or nodes?**
A: Add the appropriate directives to your job script:

```bash
# Multiple cores on a single node (shared-memory parallelism)
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8

# Multiple tasks across nodes (distributed parallelism with MPI)
#SBATCH --ntasks=16
#SBATCH --nodes=2

module load openmpi
mpirun ./my_program
```

Make sure your code actually supports the type of parallelism you're requesting. Using `--ntasks=16` has no benefit if your program is single-threaded.

---

## Software and Environment

**Q: How do I find and load software?**
A: The cluster uses the Lmod module system:

```bash
module avail              # List all available modules
module spider <name>      # Search for a specific package
module load <name>        # Load a module
module list               # Show currently loaded modules
module unload <name>      # Unload a module
module purge              # Unload all modules
```

Always load the modules you need in your job script so the environment is reproducible.

**Q: My Python/R script fails to import a library. What's wrong?**
A: The system Python/R may not have your required libraries. Best practices:

- **Python:** Create a virtual environment or conda environment in your home directory:
  ```bash
  module load python
  python -m venv ~/myenv
  source ~/myenv/bin/activate
  pip install <package>
  ```
- **R:** Install packages to a user library:
  ```r
  install.packages("package_name", lib="~/R/library")
  ```

Load the same environment in your job script that you used when installing packages.

**Q: Can I install custom software?**
A: Yes. You can install software in your home directory or project space using:

- **Conda/Mamba** for Python/R packages and bioinformatics tools
- **pip with virtual environments** for Python packages
- **Compiling from source** using `make` or `cmake`

If you need system-wide installation or a new module, email [support@ace-bioinformatics.org](mailto:support@ace-bioinformatics.org).

**Q: My script works on my laptop but fails on the cluster. Why?**
A: Common reasons:

- **Different software versions** — check with `module list` and `which <program>`. Load the correct modules.
- **Missing dependencies** — libraries available on your laptop may not be installed on the cluster.
- **Hardcoded paths** — paths like `/Users/me/data/` don't exist on the cluster. Use relative paths or update to the cluster's file system.
- **Not running through Slurm** — scripts must be submitted via `sbatch`, not run directly on the login node.

---

## Files and Storage

**Q: What's the best way to transfer files to and from the cluster?**
A: Use `scp` or `rsync`:

```bash
# Copy a file to the cluster
scp myfile.txt user@hpc.example.com:/home/user/

# Sync a directory (only transfers changed files)
rsync -avz my_project/ user@hpc.example.com:/home/user/my_project/

# Copy results back to your laptop
scp user@hpc.example.com:/home/user/results.csv ./
```

For large datasets, `rsync` is preferred because it can resume interrupted transfers and only copies files that have changed.

**Q: I'm running out of storage. What should I do?**
A: Check your usage and clean up:

```bash
# Check quota
quota -s

# Find large files in your home directory
du -sh ~/* | sort -rh | head -20

# Find large files recursively
find ~ -size +100M -exec ls -lh {} \;
```

Tips for freeing space:
- Delete intermediate files and old job outputs
- Compress large files with `gzip` or `tar`
- Move infrequently used data to an archive location
- If you genuinely need more space, request a quota increase from support

**Q: What happens to my data if my account expires?**
A: Data is retained for a limited period after account expiration. Contact support before your account expires to arrange data transfer or account renewal.

---

## Monitoring and Troubleshooting

**Q: How do I check cluster availability?**
A: Use `sinfo` to see partition and node status:

```bash
sinfo                           # Overview of all partitions
sinfo -N -l                     # Detailed per-node view
sinfo -p <partition> -o "%n %t %C %m"  # Specific partition details
```

Node states: `idle` (available), `alloc` (fully allocated), `mix` (partially allocated), `down` (unavailable).

**Q: How do I monitor a running job?**
A: Several commands are available:

```bash
squeue -u $USER                # Check job status
scontrol show job <jobid>      # Detailed job information
sacct -j <jobid> --format=JobID,Elapsed,MaxRSS,State  # Resource usage
```

For real-time monitoring of a running job, you can SSH to the node where your job is running (shown in `squeue` output) and use `top` or `htop`.

**Q: How do I test my job script before a full run?**
A: Several strategies:

1. **Use a small dataset** — run on a subset of your data with a short time limit.
2. **Interactive session** — use `salloc` to get a compute node and run commands manually.
3. **Dry run** — some tools have a `--dry-run` or `--help` flag to validate inputs.
4. **Short time limit** — submit with `--time=00:10:00` to catch errors early without waiting in the queue.

**Q: How do I run a job array?**
A: Job arrays submit many similar jobs at once, each with a unique task ID:

```bash
#!/bin/bash
#SBATCH --job-name=array_job
#SBATCH --array=1-100
#SBATCH --ntasks=1
#SBATCH --time=01:00:00
#SBATCH --output=output_%a.out

# Use the task ID to process different input files
INPUT_FILE="data/sample_${SLURM_ARRAY_TASK_ID}.fastq"
module load fastqc
fastqc $INPUT_FILE
```

You can also limit how many array tasks run simultaneously: `--array=1-100%10` runs at most 10 at a time.

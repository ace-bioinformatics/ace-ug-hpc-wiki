---
id: overview
title: Tutorials Overview
sidebar_label: Overview
---

# ACE HPC Tutorials

Welcome to the ACE HPC tutorials section. These hands-on guides are designed to take you from your first login all the way to running optimised jobs on the cluster. Each tutorial builds on the previous one, so if you are new to HPC we recommend going through them in order.

---

## Linux Basics

The ACE HPC cluster runs Linux. There is no graphical desktop — you interact entirely through a text terminal. This tutorial series bridges the gap between your personal computer and the command-line environment you will use every day on the cluster.

### [Introduction to Linux](linux-basics/linux-intro)

Aimed at researchers who are used to Windows or macOS and are encountering a Linux terminal for the first time. By the end of this tutorial you will be able to navigate the cluster confidently and automate repetitive tasks with a script.

**Concepts covered:**

- **Why HPC uses Linux** — stability, performance, and the scientific software ecosystem
- **Reading the shell prompt** — understanding what `[akatukunda@kla-ac-hpc-02 ~]$` tells you about where you are and who you are
- **The ACE filesystem layout** — the difference between `$HOME` (limited quota, for scripts and config) and `/scratch` (high-speed storage for large datasets and active job output)
- **Navigating directories** — `pwd` to find where you are, `ls -lh` to list contents with human-readable sizes, `cd` to move around, and the `~`, `.`, and `..` shortcuts
- **Absolute vs. relative paths** — when to use each and why it matters when writing job scripts
- **Creating and managing files and directories** — `mkdir -p` for nested directories in one step, `cp`, `mv`, `rm`, and using brace expansion (`{data,scripts,results,logs}`) to set up a project structure instantly
- **Viewing file contents** — `cat`, `less` for scrollable inspection, `head`/`tail` for large files, `tail -f` to watch a running job's log in real time, and `grep` to search inside files
- **File permissions** — reading the `rwxr-xr-x` permission string from `ls -l`, and using `chmod +x` to make a script executable
- **Environment variables** — what `$HOME`, `$USER`, `$SCRATCH`, and `$PATH` are, how to read them with `echo`, and how to define your own with `export`
- **Bash scripting (practical example)** — writing a complete script that loops over sample log files, detects failures with `grep`, moves failed logs to a review folder, and writes a formatted summary report; covers shebangs, variables, `for` loops, `if` statements, `>>` append redirection, and arithmetic with `(( ))`
- **Terminal habits that save time** — Tab completion, command history (`history`, Up arrow, `Ctrl+R`), stopping a running command with `Ctrl+C`, and reading manual pages with `man`

---

## Running Slurm Jobs

On a shared cluster, jobs are not run directly on the login node. Instead, you submit them to **SLURM** — the job scheduler — which queues your work and dispatches it to a compute node when resources are available. This tutorial series covers everything from your first job submission to fine-tuning performance on ACE's specialised KNL hardware.

### [SLURM Basics](running-jobs/slurm-basics)

A concise reference for the commands you will use every day once you start submitting jobs.

**Concepts covered:**

- **What SLURM does** — how it queues, schedules, and manages jobs across the cluster's shared compute resources
- **Core SLURM commands** — `sbatch` to submit a job, `squeue -u $USER` to see your active and pending jobs, `scancel` to stop a job, `scontrol show job` to inspect a specific job in detail, `sacct` to retrieve historical accounting data after a job finishes, and `sinfo` to check the state of nodes and partitions
- **A minimal working job script** — a complete example with `#SBATCH` directives for job name, output and error file paths, wall time limit, node count, task count, and memory, followed by a `module load` and a Python command
- **Best practices** — testing on small inputs first, setting realistic time limits, monitoring jobs regularly, and cleaning up with `scancel`

### [Writing Job Scripts](running-jobs/job-scripts)

A deeper guide to crafting job scripts for real scientific workloads, using bioinformatics tools as concrete examples.

**Concepts covered:**

- **The anatomy of a job script** — the shebang line, every `#SBATCH` directive explained (`--job-name`, `--output`, `--error`, `--time`, `--nodes`, `--ntasks-per-node`, `--mem`), and how SLURM reads these comments at submission time
- **Loading software modules** — using `module load` to bring a specific software version into your environment before running your tool; checking what is available with `module avail`
- **Writing the compute commands** — how the directives and the actual command connect (e.g., requesting 8 tasks and passing `-num_threads 8` to BLAST so they agree)
- **Full BLAST example** — a complete job script running `blastn` with a FASTA query against a database, with every directive annotated
- **Full HISAT2 example** — a paired-end RNA-seq alignment job showing how to set up the reference index path, read files, and output directory within the script
- **Submitting, monitoring, and reading output** — `sbatch` to submit, `squeue` to monitor, `scontrol show job` for details, and interpreting the `.out` and `.err` files after the job finishes
- **Troubleshooting hints inside the script** — checking the `.err` file, verifying file paths, and using interactive `salloc` sessions for debugging

### [Sample SLURM Job Scripts](running-jobs/sample-scripts)

A collection of ready-to-use minimal scripts you can submit immediately to verify that your environment is working and to understand the job lifecycle before committing to a long run.

**Scripts included:**

- **`list_modules.sh`** — runs `module list` on a compute node, so you can see exactly which software modules are loaded in the default environment
- **`check_disk_space.sh`** — runs `df -h` to report available disk space across all mounted filesystems visible from the compute node
- **`print_date.sh`** — prints the current timestamp, useful for verifying that your job actually ran and for measuring elapsed time between submission and execution
- **`run_python_demo.sh`** — loads a Python module and runs a one-line Hello World script, confirming that Python is accessible and your module load works correctly
- **`echo_message.sh`** — the simplest possible job: prints a custom message to the output file, useful for testing that `sbatch` can reach the cluster and write output without any dependencies

### [Using KNL Node Features](running-jobs/knl-nodes)

The ACE HPC cluster runs **Intel Xeon Phi 7250 (Knights Landing, KNL)** compute nodes. These have a radically different architecture from a standard server CPU — 68 cores, 272 hardware threads, and a unique two-tier memory system — that can dramatically accelerate the right workloads if configured correctly.

**Concepts covered:**

- **What makes KNL different** — 68 slow, highly parallel cores versus a standard node's fewer but faster cores; 272 simultaneous hardware threads; 512-bit AVX-512 vector units for wide numerical operations
- **When KNL helps and when it doesn't** — massively parallel and memory-bandwidth-hungry workloads (genome alignment, variant calling, Monte Carlo simulations, sequence assembly) benefit; single-threaded or memory-latency-bound workloads do not
- **MCDRAM — the fast on-package memory** — 16 GB of ultra-fast memory with 4–5× the bandwidth of DDR4, physically mounted on the processor; think of it as a small, blazing-fast scratchpad next to the CPU
- **MCDRAM cache mode** — MCDRAM acts as a transparent hardware-managed cache in front of DDR4; no code changes needed; the recommended default for most jobs
- **MCDRAM flat mode** — MCDRAM is exposed as a separate addressable NUMA node; use `numactl --membind=1` to explicitly place data there; only beneficial when your working set fits within 16 GB
- **MCDRAM hybrid mode** — part cache, part addressable memory; a middle-ground option for mixed workloads
- **Cluster modes** — how KNL's 68 cores are grouped for cache coherency: Quadrant (recommended default at ACE, single NUMA node, good locality), All-to-All (simple but potentially slower), SNC-2 (two NUMA sub-nodes, only useful for NUMA-aware software), SNC-4 (disabled at ACE due to boot instability)
- **The `--constraint` SLURM directive** — how to request a specific combination, e.g. `#SBATCH --constraint="quadrant&cache"`; available combinations and when to use each
- **Node reboot time** — switching memory or cluster mode requires a ~15-minute node reboot that counts against your wall time; always prefer the common `quadrant&cache` configuration to avoid delays
- **Complete HISAT2 alignment example** — a full job script using 64 cores, quadrant&cache mode, with `samtools` post-processing also parallelised
- **Complete PLINK flat-mode example** — explicit MCDRAM placement with `numactl` for a population genetics association study
- **Checking node availability and current configuration** — using `sinfo` with a grep filter to see which KNL nodes are available and what mode they are currently in

### [Troubleshooting Jobs](running-jobs/troubleshooting)

A practical guide to diagnosing the most common job failures, with specific commands for each scenario.

**Concepts covered:**

- **Job won't start** — reading `squeue` output to find the pending reason, using `scontrol show job` to see exactly why SLURM is holding a job, checking node availability with `sinfo`, and strategies for reducing resource requests to unblock a submission
- **Job fails immediately** — systematically checking the `.err` file, resolving "command not found" errors by ensuring the right module is loaded, handling segmentation faults with small interactive tests, fixing permission denied errors, and linting Bash scripts with `shellcheck`
- **Adding diagnostic output to your script** — printing timestamps with `echo "Starting at $(date)"`, dumping the environment with `env`, and enabling Bash execution tracing with `set -x`
- **Interactive debugging sessions** — using `salloc` to get a shell directly on a compute node so you can run commands manually, inspect the environment, and reproduce a failure without the batch submission cycle
- **Escalating to support** — what information to collect before emailing `support@ace-bioinformatics.org` (job ID from `sacct`, the SLURM script, the `.err` file contents, and steps already attempted)

---

## Containers

Containers let you package an application together with all of its dependencies — libraries, compilers, Python versions, system packages — into a single portable file that runs identically on your laptop and on the cluster. This eliminates the classic "it works on my machine" problem for HPC workloads.

### [What are Containers?](containers/intro)

A conceptual introduction that explains the technology and its role in reproducible HPC research.

**Concepts covered:**

- **What a container is** — a lightweight, portable package that bundles an application with its entire runtime environment; runs the same on any machine
- **Containers vs. virtual machines** — containers share the host OS kernel (fast, low overhead) while VMs include a full guest OS (heavy, slow to start); for HPC performance matters, so containers are preferred
- **Docker** — the industry-standard tool for building and distributing container images; requires root privileges and cannot run directly on shared HPC systems
- **Apptainer (formerly Singularity)** — the HPC-native container runtime used on ACE; runs as your normal user (no root required), integrates with SLURM, and can pull and run Docker images directly
- **The ACE container workflow** — build with Docker on your workstation, push to a registry (Docker Hub or GitHub Container Registry), pull with Apptainer on the cluster, run inside a SLURM job script
- **Quick start** — pulling a Python container from Docker Hub with `apptainer pull` and running a command inside it with `apptainer exec`

### [Containerize Your Code](containers/containerize-code)

A start-to-finish practical tutorial that takes a Python script and packages it into a container image ready to run on ACE HPC.

**Concepts covered:**

- **Dockerfile instructions** — `FROM` (base image), `RUN` (build-time commands), `COPY` (adding files), `ENV` (environment variables), `WORKDIR` (default directory), `CMD` (default run command)
- **Layer caching** — why you copy `requirements.txt` before your application code, and how Docker reuses cached layers to keep rebuilds fast
- **Building an image** — `docker build -t name:tag .` and how to verify the result with `docker images`
- **Testing locally** — `docker run --rm` for a clean one-shot run, passing arguments to override the default command, using bind mounts (`-v`) to write output files back to your host filesystem, and opening an interactive shell inside a container with `-it`
- **`.dockerignore`** — excluding large data files, `.git` history, and build artefacts from the build context to keep images small and builds fast
- **Pushing to a registry** — authenticating with `docker login`, tagging for Docker Hub and GitHub Container Registry, and `docker push`
- **Pulling on ACE HPC** — using `apptainer pull docker://username/image:tag` to convert a Docker image to a `.sif` file and run it on the cluster

### [Containers on HPC Clusters](containers/containers-hpc)

How to use Apptainer and your container images inside SLURM job scripts on ACE HPC.

**Concepts covered:**

- Running containers with `apptainer exec` and `apptainer run` inside batch job scripts
- Bind-mounting directories from the cluster filesystem into the container so your job can read input data and write results to `/scratch`
- GPU and MPI containers — how Apptainer passes through the host's hardware drivers so containerised applications can access cluster GPUs and high-speed networks
- Practical SLURM job script examples combining `apptainer exec` with `#SBATCH` directives

### [Advanced Build Topics](containers/advanced-builds)

Techniques for producing smaller, faster, and more portable container images.

**Concepts covered:**

- **Multi-stage builds** — using one Docker stage to compile or install dependencies and a second, minimal stage to copy only the final artefacts; significantly reduces final image size
- **Multi-architecture builds** — building images that run on both `amd64` (standard x86 cluster nodes) and `arm64` (Apple Silicon laptops) from a single `docker buildx` command

---

## Where to Start

If you are brand new to HPC, work through the tutorials in this order:

1. [Introduction to Linux](linux-basics/linux-intro) — get comfortable with the terminal
2. [SLURM Basics](running-jobs/slurm-basics) — learn how to submit jobs
3. [Writing Job Scripts](running-jobs/job-scripts) — write scripts for your own tools
4. [Containers](containers/intro) — package your software for reproducible runs on the cluster

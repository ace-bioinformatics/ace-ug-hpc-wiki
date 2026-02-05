---
id: slurm-basics
title: SLURM Basics
---

The ACE HPC Cluster uses **SLURM** (Simple Linux Utility for Resource Management) to manage job scheduling and resource allocation.

## What is SLURM?

SLURM allows you to submit, monitor, and manage jobs on the cluster. It ensures efficient use of compute resources by queueing jobs and allocating them based on availability and policy.

---

## Key Commands

| Action            | Command                                  | Description                              |
|------------------|-------------------------------------------|------------------------------------------|
| Submit job        | `sbatch script.sh`                        | Submits a job using a SLURM batch script |
| Check job queue   | `squeue -u $USER`                         | Lists your active and queued jobs        |
| Cancel job        | `scancel <job_id>`                        | Cancels the specified job                |
| Job details       | `scontrol show job <job_id>`              | Shows detailed info about a job          |
| Job history       | `sacct -j <job_id>`                       | Displays job accounting data             |
| Node status       | `sinfo`                                   | Lists node states and partition info     |
| Estimated start   | `squeue --start -u $USER`                 | Predicts job start time                  |

---

## Sample SLURM Script

```bash
#!/bin/bash
#SBATCH --job-name=myjob
#SBATCH --output=job_%j.out
#SBATCH --error=job_%j.err
#SBATCH --time=01:00:00
#SBATCH --nodes=1
#SBATCH --ntasks=4
#SBATCH --mem=8G


module load python/3.10
python my_script.py
```

### Explanation
`--job-name`: Name of your job

`--output`: Standard output file (%j is job ID)

`--error`: Standard error file

`--time`: Max wall time (HH:MM:SS)

`--nodes`: Number of nodes to use

`--ntasks`: Total number of tasks/processes

`--mem`: Memory per node

### Best Practices
- Test scripts on small inputs before scaling up.

- Use reasonable `--time` to avoid delays or early termination.

- Regularly monitor jobs with `squeue` and `sacct`.

- Cancel jobs you no longer need with `scancel`.
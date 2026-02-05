---
id: troubleshooting
title: Troubleshooting Jobs
---

When a SLURM job fails to start or runs into errors, there are several tools and techniques to help diagnose and fix the issue.

---

## Common Issues and Solutions

### 🚫 Job Won’t Start

| Check                     | Explanation                                                                 |
|--------------------------|-----------------------------------------------------------------------------|
| `squeue -u $USER`         | View your job's status in the queue. It might be waiting for resources.    |
| `scontrol show job <id>` | Get detailed info on job state, pending reasons, and requested resources.   |
| Resource Limits           | SLURM may delay your job if `--mem`, `--cpus`, or `--time` is too high.    |
| Partition Constraints     | Ensure you're submitting to the correct partition, if applicable.          |
| Node Availability         | Use `sinfo` to check if nodes are online and available.                    |

Try lowering resource requests temporarily or switching partitions (if available) to test submission.

---

### ❌ Job Fails Immediately

| Symptom                            | Solution                                                                 |
|------------------------------------|--------------------------------------------------------------------------|
| `.err` file has errors             | Always check this file for Python, R, Bash, or SLURM errors.             |
| "command not found"                | Ensure the right modules or environments are loaded in the script.       |
| Segmentation fault or core dump    | Likely a bug in your code or incompatible module—run a smaller test.     |
| Permission denied                  | Check file and directory permissions in your script or input/output paths.|
| Syntax errors                      | Review your script syntax carefully (Bash/SLURM). Use `shellcheck` to lint.|

Also confirm that any scripts are executable:  
```bash
chmod +x script.sh
```

### 📄 Helpful Debugging Tips
Add logging:
```bash
echo "Starting at $(date)"
```

Print environment:
```bash
env
```

Add `set -x` in bash scripts to trace execution.

### 🛠 Debugging with an Interactive Session
You can test interactively before submitting a batch job:

```bash
salloc --time=01:00:00 --ntasks=1 --mem=4G
```

### 📬 Still Stuck?
Contact [support@ace-bioinformatics.org](mailto:support@ace-bioinformatics.org) with:

- Job ID `sacct -j <id>`

- Your SLURM script

- Error file output

- Any steps you've already taken to debug

This helps the support team resolve the issue faster.

### Resources
[SLURM Job Exit Codes](https://slurm.schedmd.com/job_exit_code.html)

# Login Node Guidelines

Login nodes (also called head nodes) are the entry point to the ACE cluster. They are **shared among all users** and have limited resources. Running computational work on login nodes degrades the experience for everyone and may result in account suspension.

## What Login Nodes Are For

Login nodes are designed for lightweight, preparatory tasks:

- **Editing files** - Using text editors like `vim`, `nano`, or `emacs`
- **Compiling code** - Small compilation tasks (not parallel builds)
- **Managing files** - Organizing directories, moving files
- **Submitting jobs** - Using `sbatch`, `squeue`, `scancel`
- **Monitoring jobs** - Checking job status and output
- **Small file transfers** - Using `scp`, `rsync`, or `sftp`

## What NOT to Do on Login Nodes

:::danger Prohibited Activities
The following activities are **not allowed** on login nodes:
:::

### Running Research Applications

Do not launch computational software directly on login nodes:

```bash
# DON'T do this on login nodes
python my_simulation.py
matlab -batch "run_analysis"
R --vanilla < analysis.R
./my_program
```

**Instead**, submit these as SLURM batch jobs or request an interactive session.

### Parallel Compilation

Avoid resource-intensive compilation commands:

```bash
# DON'T do this on login nodes
make -j 16
ninja -j 8
```

**Instead**, use single-threaded compilation on login nodes or submit compilation as a job:

```bash
# OK on login nodes
make -j 2

# Better: submit as a job for large builds
sbatch compile_job.sh
```

### Long-Running Processes

Do not run processes that take more than a few minutes:

```bash
# DON'T leave these running on login nodes
nohup long_process &
screen -S mysession  # for computational work
```

### Excessive Resource Monitoring

Avoid polling job status too frequently:

```bash
# DON'T do this
watch -n 1 squeue -u $USER  # Every second is too frequent

# OK - check every few minutes
squeue -u $USER
```

## Using Interactive Sessions

If you need an interactive environment for testing or development, request compute resources through SLURM:

```bash
# Request an interactive session
srun --nodes=1 --ntasks=1 --time=01:00:00 --pty bash

# Or use salloc for more control
salloc --nodes=1 --ntasks=4 --time=02:00:00
```

This gives you dedicated compute resources without impacting other users.

## VSCode and IDEs

:::warning VSCode Users
Running VSCode Remote SSH on login nodes consumes significant resources and can interfere with other users. This may result in account suspension.
:::

**Recommended alternatives:**

1. **Use VSCode on your local machine** - Edit files locally and sync with the cluster
2. **Request an interactive session** - Connect VSCode to a compute node
3. **Use terminal-based editors** - `vim`, `nano`, or `emacs` on login nodes

## How We Monitor Login Node Usage

The ACE team monitors login node resource usage. Users running prohibited activities may receive:

1. An automated warning
2. Process termination
3. Temporary account suspension for repeated violations

## Summary

| Activity | Login Node | Compute Node |
|----------|------------|--------------|
| File editing | Yes | Yes |
| Job submission | Yes | No |
| Small compilation | Yes | Yes |
| Running simulations | **No** | Yes |
| Data analysis | **No** | Yes |
| Parallel builds | **No** | Yes |
| VSCode/IDEs | **No** | Yes |

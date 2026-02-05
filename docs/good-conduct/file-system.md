# File System Best Practices

The ACE cluster provides shared file systems that serve all users. Misusing storage can cause job failures, slow performance for everyone, and fill up critical system resources. Follow these guidelines to be a good citizen.

## Understanding Your Storage Areas

| File System | Purpose | Quota | Backed Up | Job I/O |
|-------------|---------|-------|-----------|---------|
| `$HOME` | Configuration, scripts | Limited | Yes | **No** |
| `$WORK` | Software, datasets | Moderate | No | Limited |
| `$SCRATCH` | Temporary job data | Large | No | **Yes** |

### Home Directory (`$HOME`)

Your home directory is for:
- Configuration files (`.bashrc`, `.profile`)
- Small scripts and source code
- Personal settings

**Do not** use `$HOME` for:
- Job input/output
- Large datasets
- Temporary files

### Work Directory (`$WORK`)

Use your work directory for:
- Installing software
- Storing original datasets
- Staging data before jobs

### Scratch Directory (`$SCRATCH`)

Use scratch for:
- All job I/O operations
- Temporary files generated during computation
- Large intermediate results

:::warning Scratch Purge Policy
Files in `$SCRATCH` that haven't been accessed for **30 days** may be automatically deleted. Do not use scratch for long-term storage.
:::

## Checking Your Quota

Monitor your storage usage regularly:

```bash
# Check quota for all file systems
quota -s

# Check disk usage in current directory
du -sh *

# Find large files in your home directory
du -ah $HOME | sort -rh | head -20
```

## What Happens When You Exceed Quota

Exceeding your quota causes:

1. **Job failures** - Jobs cannot write output files
2. **Login issues** - Shell initialization may fail if `$HOME` is full
3. **Lost work** - Running jobs may crash without saving results

## Best Practices for File Management

### Avoid Small File Accumulation

Storing thousands of small files in a single directory strains the metadata servers:

```bash
# BAD: Thousands of small files
output/
├── result_0001.txt
├── result_0002.txt
├── ... (10,000 files)
└── result_9999.txt

# BETTER: Organized subdirectories
output/
├── batch_001/
│   ├── result_0001.txt
│   └── ... (100 files)
├── batch_002/
│   └── ...
└── batch_100/
```

### Use Efficient File Formats

For large datasets, use formats designed for HPC:

```bash
# Instead of many small CSV files, consider:
- HDF5 (.h5)
- NetCDF (.nc)
- Parquet files
- Tar archives
```

### Clean Up Regularly

```bash
# Remove old temporary files
find $SCRATCH -name "*.tmp" -mtime +7 -delete

# Archive completed project data
tar -czvf project_archive.tar.gz project_dir/
rm -rf project_dir/
```

### Run Jobs in Scratch

Always direct job I/O to `$SCRATCH`:

```bash
#!/bin/bash
#SBATCH --job-name=my_job
#SBATCH --output=$SCRATCH/jobs/%j.out

# Work in scratch
cd $SCRATCH
mkdir -p job_$SLURM_JOB_ID
cd job_$SLURM_JOB_ID

# Run your computation
./my_program

# Copy important results back
cp final_results.dat $WORK/results/
```

## I/O Optimization Tips

### Minimize File Operations

```python
# BAD: Opening file in a loop
for i in range(10000):
    with open('output.txt', 'a') as f:
        f.write(f"Result {i}\n")

# GOOD: Open once, write many times
with open('output.txt', 'w') as f:
    for i in range(10000):
        f.write(f"Result {i}\n")
```

### Batch Checkpoint Files

Instead of writing checkpoints every iteration:

```bash
# BAD: Checkpoint every step
checkpoint_001.dat
checkpoint_002.dat
... (thousands of files)

# GOOD: Checkpoint at intervals, overwrite previous
checkpoint_latest.dat
checkpoint_backup.dat
```

### Avoid Simultaneous Large I/O Jobs

If you plan to run many I/O-intensive jobs simultaneously, contact support first to discuss your workflow.

## Summary

| Do | Don't |
|---|---|
| Run jobs in `$SCRATCH` | Run I/O-intensive jobs in `$HOME` |
| Check quota regularly | Ignore quota warnings |
| Archive small files | Store thousands of tiny files |
| Clean up after jobs | Leave temporary files indefinitely |
| Use HDF5/NetCDF for large data | Use thousands of small text files |

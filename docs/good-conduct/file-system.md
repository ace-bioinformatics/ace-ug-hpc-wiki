# File System Best Practices

The ACE cluster provides shared file systems that serve all users. Misusing storage can cause job failures, slow performance for everyone, and fill up critical system resources. Follow these guidelines to be a good citizen.

## Understanding Your Storage Areas

| File System | Purpose | Default Quota | Max Quota |
|-------------|---------|---------------|-----------|
| `$HOME` | Scripts, configs, software, datasets, job output | 50 GB | 200 GB |

### Home Directory (`$HOME`)

Your home directory is your primary workspace on ACE HPC. It is for:
- Configuration files (`.bashrc`, `.profile`)
- Scripts and source code
- Personal settings
- Software installations (`~/.local/`)
- Datasets and job input/output (within quota)

Organise your work into subdirectories to keep things manageable:

```bash
$HOME/
├── projects/       # research projects
│   └── my_project/
│       ├── data/
│       ├── scripts/
│       └── results/
├── jobs/           # SLURM job output logs
└── software/       # locally compiled tools
```

:::warning Monitor your quota
Your home directory has a **50 GB** default quota. Exceeding this causes job failures and may prevent login. Check your usage regularly with `du -sh $HOME` and clean up files you no longer need.
:::

## Storage Quotas

### User Home Directories

| Quota Type | Allocation |
|------------|-----------|
| Default | 50 GB |
| Maximum (with approved request) | 200 GB |

### Project Directories

Project storage is available for groups with larger data requirements:

| Quota Type | Allocation |
|------------|-----------|
| Default | 1 TB |
| Maximum (with approved request) | 5 TB |

To request a project directory, contact [support@ace-bioinformatics.org](mailto:support@ace-bioinformatics.org) with a justification for the storage requirement, the project details, estimated data sizes, and the duration for which the storage is needed.

### Requesting Additional Storage

If your work requires more than the default quota, submit a request to [support@ace-bioinformatics.org](mailto:support@ace-bioinformatics.org) including:

- Justification for the additional storage requirement
- Project details and estimated data sizes
- Duration for which the additional storage is needed

Requests are reviewed by the HPC administration team and forwarded to the Centre Director for final approval. You will be notified of the decision within two weeks.

## Checking Your Quota

Monitor your storage usage regularly. You will receive an alert when your usage reaches 90% of your quota:

```bash
# Check your quota and current usage
quota -s

# Check disk usage in current directory
du -sh *

# Find large files in your home directory
du -ah $HOME | sort -rh | head -20
```

If you exceed your quota, write access will be suspended until you reduce your usage below the allocated limit.

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
# Remove old temporary files from your home directory
find $HOME -name "*.tmp" -mtime +7 -delete

# Archive completed project data to free space
tar -czvf project_archive.tar.gz project_dir/
rm -rf project_dir/
```

### Organise Job Output

Direct job output to a dedicated directory within `$HOME`:

```bash
#!/bin/bash
#SBATCH --job-name=my_job
#SBATCH --output=$HOME/jobs/%j.out

# Create a job-specific working directory
mkdir -p $HOME/jobs/job_$SLURM_JOB_ID
cd $HOME/jobs/job_$SLURM_JOB_ID

# Run your computation
./my_program

# Results stay in $HOME/jobs/job_$SLURM_JOB_ID
# Clean up when you no longer need them
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

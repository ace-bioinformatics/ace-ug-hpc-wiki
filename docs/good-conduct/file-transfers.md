# File Transfer Guide

Transferring files to and from the ACE cluster requires care to avoid overloading the login nodes and shared file systems. Follow these guidelines for efficient, responsible file transfers.

## Transfer Methods

| Method | Best For | Command |
|--------|----------|---------|
| `scp` | Single files, small directories | `scp file user@ace:~/` |
| `rsync` | Large directories, resumable transfers | `rsync -avz dir/ user@ace:~/dir/` |
| `sftp` | Interactive transfers | `sftp user@ace` |

## Small File Transfer Best Practices

### Archive Before Transferring

Transferring thousands of small files individually is extremely slow and stresses the file system. Always archive first:

```bash
# On your local machine, before transferring:
tar -czvf my_data.tar.gz my_data_directory/

# Transfer the single archive
scp my_data.tar.gz username@ace.hpc.ac.ug:$SCRATCH/

# On ACE, extract the archive
tar -xzvf my_data.tar.gz
```

### Why Archives Are Better

| Approach | 10,000 files @ 1KB each | Time |
|----------|------------------------|------|
| Individual transfers | 10,000 operations | Very slow |
| Single tar archive | 1 operation | Fast |

Each file operation involves metadata overhead. Archiving eliminates this overhead.

## Large File Transfers

### Using rsync (Recommended)

`rsync` is the best tool for large transfers because it:
- Resumes interrupted transfers
- Only transfers changed portions of files
- Compresses data during transfer

```bash
# Basic rsync with compression
rsync -avz local_dir/ username@ace.hpc.ac.ug:$WORK/local_dir/

# Resume an interrupted transfer
rsync -avz --partial local_dir/ username@ace.hpc.ac.ug:$WORK/local_dir/

# Show progress for large files
rsync -avz --progress large_file.dat username@ace.hpc.ac.ug:$SCRATCH/
```

### rsync Options Explained

| Flag | Meaning |
|------|---------|
| `-a` | Archive mode (preserves permissions, timestamps) |
| `-v` | Verbose output |
| `-z` | Compress during transfer |
| `--partial` | Keep partially transferred files |
| `--progress` | Show transfer progress |

### Using scp

For simple, one-time transfers:

```bash
# Single file
scp myfile.dat username@ace.hpc.ac.ug:$SCRATCH/

# Directory (recursive)
scp -r my_directory/ username@ace.hpc.ac.ug:$WORK/

# With compression
scp -C large_file.dat username@ace.hpc.ac.ug:$SCRATCH/
```

## Concurrent Transfer Limits

:::warning Transfer Limits
Limit yourself to **2-3 concurrent transfer sessions** maximum. More simultaneous transfers can overload the login nodes and network.
:::

```bash
# BAD: Many parallel transfers
for i in {1..10}; do
    scp file_$i.dat ace:$SCRATCH/ &
done

# GOOD: Sequential or limited parallel
for file in *.dat; do
    scp "$file" ace:$SCRATCH/
done

# Or archive everything first
tar -cvf data.tar *.dat
scp data.tar ace:$SCRATCH/
```

## Transferring Results Back

### Archive Results Before Downloading

```bash
# On ACE, prepare results for download
cd $SCRATCH/job_output
tar -czvf results.tar.gz important_results/

# From your local machine
scp username@ace.hpc.ac.ug:$SCRATCH/job_output/results.tar.gz ./
```

### Selective Sync with rsync

Only download changed files:

```bash
# Sync results to local machine (only new/changed files)
rsync -avz username@ace.hpc.ac.ug:$WORK/project/results/ ./local_results/
```

## Transfer Tips by Scenario

### Scenario: Upload Code Repository

```bash
# Use rsync with exclusions
rsync -avz --exclude='.git' --exclude='node_modules' \
    my_project/ username@ace.hpc.ac.ug:$WORK/my_project/
```

### Scenario: Upload Large Dataset

```bash
# For very large files, use rsync with progress
rsync -avz --progress large_dataset.hdf5 \
    username@ace.hpc.ac.ug:$SCRATCH/data/
```

### Scenario: Many Small Output Files

```bash
# On ACE, archive outputs before download
cd $SCRATCH/simulation
tar -czvf outputs.tar.gz output_dir/

# Download the archive
# (from local machine)
scp username@ace.hpc.ac.ug:$SCRATCH/simulation/outputs.tar.gz ./
tar -xzvf outputs.tar.gz
```

### Scenario: Regular Sync Between Machines

```bash
# Create a sync script
#!/bin/bash
rsync -avz --delete \
    --exclude='*.tmp' \
    --exclude='__pycache__' \
    ~/project/ username@ace.hpc.ac.ug:$WORK/project/
```

## What to Avoid

| Don't | Do Instead |
|-------|------------|
| Transfer 10,000+ small files individually | Archive first, then transfer |
| Run 10+ concurrent transfers | Limit to 2-3 transfers |
| Transfer to `$HOME` | Transfer to `$SCRATCH` or `$WORK` |
| Use `cp` over network mounts | Use `scp` or `rsync` |
| Transfer during peak hours if avoidable | Schedule large transfers off-peak |

## Checking Transfer Success

```bash
# Verify file integrity with checksums
# On source machine
md5sum large_file.dat > checksum.md5

# Transfer both
scp large_file.dat checksum.md5 ace:$SCRATCH/

# On ACE, verify
cd $SCRATCH
md5sum -c checksum.md5
```

## Summary

1. **Archive small files** before transferring
2. **Use rsync** for large or repeated transfers
3. **Limit concurrent transfers** to 2-3 sessions
4. **Transfer to `$SCRATCH`** for job data, `$WORK` for persistent data
5. **Verify integrity** for critical data

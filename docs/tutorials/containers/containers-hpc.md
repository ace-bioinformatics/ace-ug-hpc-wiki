---
id: containers-hpc
title: Containers on High Performance Compute Clusters
sidebar_label: Containers on HPC
sidebar_position: 4
---

# Containers on High Performance Compute Clusters

This tutorial covers running containerized workloads on ACE HPC. You'll learn how to use Apptainer to pull and run containers, write SLURM job scripts, manage data with bind mounts, and optimize performance on the cluster.

## Learning Objectives

After completing this tutorial, you will be able to:

- Use Apptainer to pull and convert Docker images
- Execute commands inside containers on ACE HPC
- Write SLURM job scripts for containerized workflows
- Manage input/output data with bind mounts
- Run GPU and MPI jobs using containers
- Troubleshoot common container issues on HPC

---

## Getting Started with Apptainer

### Loading the Apptainer Module

On ACE HPC, Apptainer is available as a module:

```bash
# Load the Apptainer module
module load apptainer

# Verify installation
apptainer --version
```

**Expected output:**
```
apptainer version 1.2.x
```

### Pulling Container Images

Apptainer can pull images from Docker Hub and other registries:

```bash
# Pull from Docker Hub
apptainer pull docker://python:3.11-slim

# This creates: python_3.11-slim.sif
```

The `.sif` (Singularity Image Format) file is Apptainer's native format - a single, portable file containing the entire container.

### Pull Options

```bash
# Specify output filename
apptainer pull python.sif docker://python:3.11-slim

# Pull to a specific directory
apptainer pull ~/containers/pytorch.sif docker://pytorch/pytorch:2.0.1-cuda11.7-cudnn8-runtime

# Force re-download (ignore cache)
apptainer pull --force python.sif docker://python:3.11-slim
```

### Managing the Cache

Apptainer caches downloaded layers:

```bash
# View cache location and size
apptainer cache list

# Clean the cache (free disk space)
apptainer cache clean
```

---

## Exercise 1: Pull Your First Image

```bash
# Create a container directory
mkdir -p ~/containers
cd ~/containers

# Load Apptainer
module load apptainer

# Pull Python image
apptainer pull docker://python:3.11-slim

# Verify
ls -lh python_3.11-slim.sif
```

---

## Running Containers

### Three Ways to Run

| Command | Purpose | Example |
|---------|---------|---------|
| `apptainer exec` | Run a specific command | `apptainer exec python.sif python script.py` |
| `apptainer run` | Run the default command | `apptainer run python.sif` |
| `apptainer shell` | Interactive shell | `apptainer shell python.sif` |

### Using `exec` (Most Common)

```bash
# Run a Python command
apptainer exec python_3.11-slim.sif python --version

# Run a Python script
apptainer exec python_3.11-slim.sif python analysis.py

# Run with arguments
apptainer exec python_3.11-slim.sif python train.py --epochs 100 --batch-size 32
```

### Using `shell` (Interactive)

```bash
# Start interactive shell
apptainer shell python_3.11-slim.sif

# You'll see a new prompt:
Apptainer> python --version
Apptainer> pip list
Apptainer> exit
```

### Using `run` (Default Action)

```bash
# Run the container's default command
apptainer run python_3.11-slim.sif
```

---

## Exercise 2: Run Commands in Containers

```bash
cd ~/containers

# Execute a Python one-liner
apptainer exec python_3.11-slim.sif python -c "
import sys
print(f'Python version: {sys.version}')
print(f'Executable: {sys.executable}')
"

# Check what packages are available
apptainer exec python_3.11-slim.sif pip list

# Start an interactive session
apptainer shell python_3.11-slim.sif
# Try: python, pip list, cat /etc/os-release, exit
```

---

## Understanding Bind Mounts

By default, Apptainer automatically binds (mounts) several directories:

| Host Directory | Container Path | Access |
|----------------|----------------|--------|
| `$HOME` | `$HOME` | Read/Write |
| `$PWD` | `$PWD` | Read/Write |
| `/tmp` | `/tmp` | Read/Write |

This means your home directory and current working directory are accessible inside the container.

### Verifying Default Binds

```bash
# Check what you can access
apptainer exec python_3.11-slim.sif ls ~

# Your current directory is available
echo "Hello from host" > test.txt
apptainer exec python_3.11-slim.sif cat test.txt
```

### Custom Bind Mounts

Use `--bind` or `-B` to mount additional directories:

```bash
# Mount a data directory
apptainer exec --bind /data/shared:/data python_3.11-slim.sif ls /data

# Mount read-only
apptainer exec --bind /data/shared:/data:ro python_3.11-slim.sif ls /data

# Mount multiple directories
apptainer exec \
    --bind /data/input:/input:ro \
    --bind /scratch/$USER:/output \
    python_3.11-slim.sif python process.py

# Mount with different path inside container
apptainer exec --bind /data/project/raw_reads:/reads python_3.11-slim.sif ls /reads
```

---

## SLURM Job Scripts for Containers

### Basic Container Job

```bash
#!/bin/bash
#SBATCH --job-name=container-job
#SBATCH --output=%x_%j.out
#SBATCH --error=%x_%j.err
#SBATCH --time=01:00:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=8G

# Load Apptainer
module load apptainer

# Define paths
CONTAINER=~/containers/python_3.11-slim.sif
SCRIPT=~/project/analysis.py

# Run the container
apptainer exec $CONTAINER python $SCRIPT
```

### Job with Data Binding

```bash
#!/bin/bash
#SBATCH --job-name=data-analysis
#SBATCH --output=%x_%j.out
#SBATCH --error=%x_%j.err
#SBATCH --time=04:00:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G

module load apptainer

# Paths
CONTAINER=~/containers/datascience.sif
INPUT_DIR=/data/shared/project/raw_data
OUTPUT_DIR=/scratch/$USER/results_$SLURM_JOB_ID

# Create output directory
mkdir -p $OUTPUT_DIR

# Run with bind mounts
apptainer exec \
    --bind $INPUT_DIR:/input:ro \
    --bind $OUTPUT_DIR:/output \
    $CONTAINER python /app/analyze.py \
        --input /input \
        --output /output

# Copy results to permanent storage
cp -r $OUTPUT_DIR /data/shared/project/results/
```

---

## Exercise 3: Submit a Container Job

Create a complete job workflow:

```bash
mkdir -p ~/container-job
cd ~/container-job
```

Create a Python script:

```bash
cat > process.py << 'EOF'
#!/usr/bin/env python3
"""Sample processing script."""

import os
import json
from datetime import datetime

def main():
    output_dir = os.environ.get('OUTPUT_DIR', '/output')
    job_id = os.environ.get('SLURM_JOB_ID', 'local')

    print(f"Job ID: {job_id}")
    print(f"Output directory: {output_dir}")
    print(f"Processing started at: {datetime.now()}")

    # Simulate some work
    results = {
        'job_id': job_id,
        'timestamp': datetime.now().isoformat(),
        'status': 'completed',
        'data': list(range(10))
    }

    # Save results
    output_file = os.path.join(output_dir, f'results_{job_id}.json')
    with open(output_file, 'w') as f:
        json.dump(results, f, indent=2)

    print(f"Results saved to: {output_file}")
    print(f"Processing completed at: {datetime.now()}")

if __name__ == "__main__":
    main()
EOF
```

Create the job script:

```bash
cat > job.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=container-test
#SBATCH --output=%x_%j.out
#SBATCH --error=%x_%j.err
#SBATCH --time=00:10:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --mem=2G

module load apptainer

# Setup
WORK_DIR=~/container-job
OUTPUT_DIR=$WORK_DIR/output_$SLURM_JOB_ID
mkdir -p $OUTPUT_DIR

# Run container
apptainer exec \
    --env OUTPUT_DIR=$OUTPUT_DIR \
    ~/containers/python_3.11-slim.sif \
    python $WORK_DIR/process.py

# Show results
echo "Output files:"
ls -la $OUTPUT_DIR/
EOF
```

Submit and monitor:

```bash
sbatch job.sh
squeue -u $USER
# After completion:
cat container-test_*.out
```

---

## GPU Jobs with Containers

### Enabling GPU Access

Use the `--nv` flag to enable NVIDIA GPU access:

```bash
apptainer exec --nv pytorch-gpu.sif python train.py
```

### GPU SLURM Job

```bash
#!/bin/bash
#SBATCH --job-name=gpu-training
#SBATCH --output=%x_%j.out
#SBATCH --error=%x_%j.err
#SBATCH --time=08:00:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --gres=gpu:1
#SBATCH --partition=gpu

module load apptainer

CONTAINER=~/containers/pytorch-gpu.sif

# Run with GPU access
apptainer exec --nv \
    --bind /data/datasets:/data:ro \
    --bind /scratch/$USER:/output \
    $CONTAINER python train.py \
        --data-dir /data \
        --output-dir /output \
        --epochs 100
```

### Multi-GPU Job

```bash
#!/bin/bash
#SBATCH --job-name=multi-gpu
#SBATCH --output=%x_%j.out
#SBATCH --error=%x_%j.err
#SBATCH --time=24:00:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=32
#SBATCH --mem=128G
#SBATCH --gres=gpu:4
#SBATCH --partition=gpu

module load apptainer

apptainer exec --nv ~/containers/pytorch-gpu.sif python -c "
import torch
print(f'GPUs available: {torch.cuda.device_count()}')
for i in range(torch.cuda.device_count()):
    print(f'  GPU {i}: {torch.cuda.get_device_name(i)}')
"

# Run distributed training
apptainer exec --nv ~/containers/pytorch-gpu.sif \
    torchrun --nproc_per_node=4 train_distributed.py
```

---

## Exercise 4: GPU Container Job

```bash
mkdir -p ~/gpu-job
cd ~/gpu-job
```

Create a GPU test script:

```bash
cat > gpu_test.py << 'EOF'
#!/usr/bin/env python3
"""Test GPU availability."""

import torch
import os

print("=" * 60)
print("GPU Container Test")
print("=" * 60)
print(f"Job ID: {os.environ.get('SLURM_JOB_ID', 'N/A')}")
print(f"PyTorch: {torch.__version__}")
print(f"CUDA available: {torch.cuda.is_available()}")

if torch.cuda.is_available():
    print(f"CUDA version: {torch.version.cuda}")
    print(f"GPU count: {torch.cuda.device_count()}")
    for i in range(torch.cuda.device_count()):
        props = torch.cuda.get_device_properties(i)
        print(f"  GPU {i}: {props.name} ({props.total_memory/1e9:.1f} GB)")

    # Quick test
    x = torch.randn(1000, 1000, device='cuda')
    y = torch.matmul(x, x)
    print(f"\nGPU computation test: PASSED")
else:
    print("\nNo GPU detected. Did you use --nv flag?")
print("=" * 60)
EOF
```

Create job script:

```bash
cat > gpu_job.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=gpu-test
#SBATCH --output=%x_%j.out
#SBATCH --error=%x_%j.err
#SBATCH --time=00:15:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G
#SBATCH --gres=gpu:1
#SBATCH --partition=gpu

module load apptainer

apptainer exec --nv ~/containers/pytorch-gpu.sif python ~/gpu-job/gpu_test.py
EOF
```

---

## MPI Jobs with Containers

### Running MPI Containers

```bash
#!/bin/bash
#SBATCH --job-name=mpi-container
#SBATCH --output=%x_%j.out
#SBATCH --error=%x_%j.err
#SBATCH --time=02:00:00
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=16
#SBATCH --mem=64G

module load apptainer
module load openmpi

# Use host MPI to launch container processes
mpirun -np $SLURM_NTASKS apptainer exec ~/containers/mpi-app.sif /app/parallel_sim
```

### Hybrid MPI + OpenMP

```bash
#!/bin/bash
#SBATCH --job-name=hybrid-mpi
#SBATCH --output=%x_%j.out
#SBATCH --error=%x_%j.err
#SBATCH --time=04:00:00
#SBATCH --nodes=4
#SBATCH --ntasks-per-node=4
#SBATCH --cpus-per-task=8
#SBATCH --mem=64G

module load apptainer
module load openmpi

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

mpirun -np $SLURM_NTASKS apptainer exec \
    --env OMP_NUM_THREADS=$OMP_NUM_THREADS \
    ~/containers/hybrid-app.sif /app/hybrid_code
```

---

## Array Jobs

Process multiple inputs efficiently:

```bash
#!/bin/bash
#SBATCH --job-name=array-container
#SBATCH --output=logs/%x_%A_%a.out
#SBATCH --error=logs/%x_%A_%a.err
#SBATCH --time=01:00:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=8G
#SBATCH --array=1-100%20

# %20 limits concurrent jobs to 20

module load apptainer

# Use array task ID to select input
INPUT_FILE=/data/inputs/sample_${SLURM_ARRAY_TASK_ID}.csv
OUTPUT_FILE=/data/outputs/result_${SLURM_ARRAY_TASK_ID}.csv

echo "Array task $SLURM_ARRAY_TASK_ID: Processing $INPUT_FILE"

apptainer exec --bind /data:/data ~/containers/analysis.sif \
    python process.py --input $INPUT_FILE --output $OUTPUT_FILE
```

---

## Environment Variables

### Passing Variables to Containers

```bash
# Using --env
apptainer exec --env MY_VAR=value container.sif printenv MY_VAR

# Multiple variables
apptainer exec \
    --env VAR1=value1 \
    --env VAR2=value2 \
    container.sif python script.py

# From host environment using APPTAINERENV_ prefix
export APPTAINERENV_MY_VAR="hello"
apptainer exec container.sif printenv MY_VAR
```

### SLURM Variables in Containers

```bash
#!/bin/bash
#SBATCH ...

module load apptainer

# SLURM variables are automatically available
apptainer exec container.sif python -c "
import os
print(f'Job ID: {os.environ.get(\"SLURM_JOB_ID\")}')
print(f'Node: {os.environ.get(\"SLURM_NODEID\")}')
print(f'CPUs: {os.environ.get(\"SLURM_CPUS_PER_TASK\")}')
"
```

### Clean Environment

Start with minimal environment:

```bash
# Ignore host environment variables
apptainer exec --cleanenv container.sif env
```

---

## Performance Optimization

### Use Local Scratch for I/O

```bash
#!/bin/bash
#SBATCH ...

module load apptainer

# Copy data to fast local storage
LOCAL_DATA=/scratch/$USER/job_$SLURM_JOB_ID
mkdir -p $LOCAL_DATA
cp -r /data/input/* $LOCAL_DATA/

# Run with local data
apptainer exec --bind $LOCAL_DATA:/data container.sif python process.py

# Copy results back
cp -r $LOCAL_DATA/results /data/output/
rm -rf $LOCAL_DATA
```

### Store Containers on Fast Storage

```bash
# Store frequently-used containers on scratch
cp ~/containers/large-container.sif /scratch/$USER/containers/

# Use from scratch
apptainer exec /scratch/$USER/containers/large-container.sif python app.py
```

### Avoid Unnecessary Bind Mounts

```bash
# Bad: Many small mounts
apptainer exec -B /d1:/d1 -B /d2:/d2 -B /d3:/d3 ...

# Good: Mount parent directory
apptainer exec -B /data:/data container.sif ...
```

---

## Troubleshooting

### Common Issues and Solutions

**Container not found:**
```bash
# Check the path
ls -la ~/containers/mycontainer.sif

# Use absolute path
apptainer exec /home/user/containers/mycontainer.sif ...
```

**Permission denied:**
```bash
# Check file permissions
ls -la container.sif

# Make readable
chmod +r container.sif
```

**"Command not found" inside container:**
```bash
# Check what's available
apptainer exec container.sif ls /usr/bin/
apptainer exec container.sif which python

# Use full path
apptainer exec container.sif /usr/local/bin/python
```

**GPU not found:**
```bash
# Did you forget --nv?
apptainer exec --nv container.sif nvidia-smi

# Check SLURM allocation
echo $CUDA_VISIBLE_DEVICES
```

**Out of disk space during pull:**
```bash
# Set cache to scratch
export APPTAINER_CACHEDIR=/scratch/$USER/.apptainer
mkdir -p $APPTAINER_CACHEDIR
apptainer pull docker://large/image
```

**Job runs out of memory:**
```bash
# Increase SLURM memory allocation
#SBATCH --mem=64G

# Or limit container memory
apptainer exec --memory 60G container.sif python memory_heavy.py
```

---

## Job Script Templates

### Template: Standard CPU Job

```bash
#!/bin/bash
#SBATCH --job-name=cpu-container
#SBATCH --output=%x_%j.out
#SBATCH --error=%x_%j.err
#SBATCH --time=04:00:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G

module load apptainer

CONTAINER=~/containers/myapp.sif
WORK_DIR=$(pwd)

apptainer exec \
    --bind $WORK_DIR:/work \
    --pwd /work \
    $CONTAINER python script.py --input data.csv --output results.csv
```

### Template: GPU Deep Learning

```bash
#!/bin/bash
#SBATCH --job-name=dl-training
#SBATCH --output=%x_%j.out
#SBATCH --error=%x_%j.err
#SBATCH --time=24:00:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=16
#SBATCH --mem=64G
#SBATCH --gres=gpu:4
#SBATCH --partition=gpu

module load apptainer

CONTAINER=~/containers/pytorch-gpu.sif
DATA_DIR=/data/datasets
OUTPUT_DIR=/scratch/$USER/training_$SLURM_JOB_ID

mkdir -p $OUTPUT_DIR

apptainer exec --nv \
    --bind $DATA_DIR:/data:ro \
    --bind $OUTPUT_DIR:/output \
    $CONTAINER python train.py \
        --data /data \
        --output /output \
        --epochs 100 \
        --batch-size 64

# Save final model
cp $OUTPUT_DIR/best_model.pt /data/models/
```

### Template: Bioinformatics Pipeline

```bash
#!/bin/bash
#SBATCH --job-name=bio-pipeline
#SBATCH --output=%x_%j.out
#SBATCH --error=%x_%j.err
#SBATCH --time=12:00:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=16
#SBATCH --mem=64G

module load apptainer

# Containers
FASTQC=~/containers/fastqc.sif
BWA=~/containers/bwa.sif
SAMTOOLS=~/containers/samtools.sif

# Paths
READS=/data/raw_reads
REF=/data/reference/genome.fa
OUT=/scratch/$USER/pipeline_$SLURM_JOB_ID

mkdir -p $OUT/{qc,aligned,sorted}

echo "Step 1: Quality control"
apptainer exec --bind /data:/data,$OUT:/out $FASTQC \
    fastqc $READS/*.fastq.gz -o /out/qc -t $SLURM_CPUS_PER_TASK

echo "Step 2: Alignment"
apptainer exec --bind /data:/data,$OUT:/out $BWA \
    bwa mem -t $SLURM_CPUS_PER_TASK $REF $READS/*_R1.fastq.gz $READS/*_R2.fastq.gz \
    > $OUT/aligned/sample.sam

echo "Step 3: Sort and index"
apptainer exec --bind $OUT:/out $SAMTOOLS \
    samtools sort -@ $SLURM_CPUS_PER_TASK /out/aligned/sample.sam -o /out/sorted/sample.bam

apptainer exec --bind $OUT:/out $SAMTOOLS \
    samtools index /out/sorted/sample.bam

echo "Pipeline complete!"
ls -lh $OUT/sorted/
```

---

## Quick Reference

### Essential Commands

```bash
# Load module
module load apptainer

# Pull image
apptainer pull docker://image:tag
apptainer pull output.sif docker://image:tag

# Run commands
apptainer exec image.sif command
apptainer exec --nv image.sif command          # With GPU
apptainer exec --bind /src:/dst image.sif cmd  # With bind mount

# Interactive
apptainer shell image.sif

# Inspect
apptainer inspect image.sif
```

### Common Flags

| Flag | Purpose |
|------|---------|
| `--nv` | Enable NVIDIA GPU |
| `--bind`, `-B` | Bind mount directories |
| `--env`, `-e` | Set environment variable |
| `--cleanenv` | Start with clean environment |
| `--pwd` | Set working directory |
| `--contain` | Minimal isolation |
| `--writable-tmpfs` | Writable overlay |

---

## Summary

In this tutorial, you learned:

- How to use Apptainer to pull and run Docker images on ACE HPC
- How to write SLURM job scripts for containerized workloads
- How to manage data with bind mounts
- How to run GPU and MPI jobs with containers
- Performance optimization strategies
- Troubleshooting common issues

---

## Additional Resources

- [Apptainer User Guide](https://apptainer.org/docs/user/latest/)
- [ACE HPC SLURM Basics](../../running-jobs/slurm-basics)
- [Sample SLURM Scripts](../../running-jobs/sample-scripts)
- [ACE HPC Support](../../support/contact)

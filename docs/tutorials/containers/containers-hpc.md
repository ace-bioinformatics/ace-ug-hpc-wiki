---
id: containers-hpc
title: Containers on HPC Clusters
sidebar_label: Containers on HPC
sidebar_position: 4
---

# Containers on HPC Clusters

This tutorial covers deploying containers on ACE HPC using Apptainer and SLURM. You'll learn how to pull and run Docker images, write batch job scripts for containerized workloads, and run MPI and GPU jobs inside containers.

## How Apptainer Differs from Docker

If you've followed the previous tutorials, you've been using Docker on your workstation. Apptainer works differently in a few important ways:

| | Docker | Apptainer |
|-|--------|-----------|
| **Privileges** | Requires root (admin) access | Runs as your normal user |
| **Image format** | Layered images stored in a daemon | Single `.sif` file you can copy and move |
| **Directory mounts** | Must be explicitly specified with `-v` | Automatically mounts `$HOME`, `$PWD`, and `/tmp` |
| **User identity** | Runs as root inside the container by default | Runs as *you* — same UID, same permissions |
| **Where to build** | On your workstation | Pull pre-built images (no `sudo` on HPC) |

The key takeaway: you **build** containers with Docker on your workstation (where you have root), then **run** them with Apptainer on the cluster (where you don't).

## Introduction to Apptainer

### Loading the Module

Apptainer is available as a module on ACE HPC. Start an interactive session on a compute node before working with containers — don't pull or run containers on the login node:

```bash
# Request an interactive session
salloc --ntasks=1 --mem=4G --time=01:00:00

# Load Apptainer
module load apptainer

# Verify
apptainer --version
```

### Pulling an Image

Apptainer can pull any Docker image and convert it to its native `.sif` format:

```bash
# Pull the Monte Carlo image we built in previous tutorials
apptainer pull docker://yourusername/monte-carlo:0.1

# This creates: monte-carlo_0.1.sif
ls -lh monte-carlo_0.1.sif
```

The `.sif` file is a single, portable file containing the entire container. You can copy it, move it, share it — it's just a file.

To pull to a specific directory or with a custom filename:

```bash
mkdir -p ~/containers
apptainer pull ~/containers/monte-carlo.sif docker://yourusername/monte-carlo:0.1
```

:::note Cache management
Apptainer caches downloaded layers in `~/.apptainer/cache`. If your home directory runs low on space, redirect the cache to scratch storage:
```bash
export APPTAINER_CACHEDIR=/scratch/$USER/.apptainer
mkdir -p $APPTAINER_CACHEDIR
```
Clean the cache with `apptainer cache clean`.
:::

### Running Containers

Apptainer has three commands for running containers:

**`apptainer exec`** — Run a specific command inside the container. This is what you'll use most often:

```bash
# Run the Monte Carlo simulation
apptainer exec monte-carlo_0.1.sif python /app/estimate_pi.py 5000000

# Check what Python version is inside the container
apptainer exec monte-carlo_0.1.sif python --version

# List installed packages
apptainer exec monte-carlo_0.1.sif pip list
```

**`apptainer shell`** — Start an interactive shell inside the container for exploration and debugging:

```bash
apptainer shell monte-carlo_0.1.sif
Apptainer> python --version
Apptainer> ls /app/
Apptainer> cat /etc/os-release
Apptainer> exit
```

Notice that inside the shell, you're still *you* — your home directory is accessible, your files are there, and you have the same permissions as outside. This is different from Docker, where you'd typically be root.

**`apptainer run`** — Execute the container's default command (its `CMD` or `ENTRYPOINT`):

```bash
apptainer run monte-carlo_0.1.sif
```

### Bind Mounts

Apptainer automatically mounts your home directory, current working directory, and `/tmp`. This means scripts and data in those locations are directly accessible inside the container without any extra flags.

For directories outside these defaults, use `--bind` (or `-B`):

```bash
# Mount a shared data directory as read-only, and a scratch directory for output
apptainer exec \
    --bind /data/shared/project:/input:ro \
    --bind /scratch/$USER/results:/output \
    monte-carlo_0.1.sif python /app/estimate_pi.py 10000000 --output /output/results.json
```

The syntax is `--bind host_path:container_path[:ro]`. The `:ro` suffix makes the mount read-only, which is good practice for input data so you don't accidentally overwrite it.

### Environment Variables

```bash
# Pass environment variables to the container
apptainer exec --env MY_VAR=hello monte-carlo_0.1.sif printenv MY_VAR

# SLURM variables ($SLURM_JOB_ID, $SLURM_CPUS_PER_TASK, etc.) are
# automatically available inside the container.

# Use --cleanenv to start with a minimal environment, ignoring host variables
# that might conflict with the container's software:
apptainer exec --cleanenv monte-carlo_0.1.sif python /app/estimate_pi.py
```

## SLURM Batch Jobs

On an HPC cluster, you don't run jobs interactively — you write a batch script that specifies the resources you need and the commands to run, then submit it to the scheduler. The scheduler runs your job when resources become available.

### The Example: Monte Carlo Batch Job

Create a file called `monte-carlo.slurm`:

```bash
#!/bin/bash
#SBATCH --job-name=monte-carlo
#SBATCH --output=monte-carlo_%j.out
#SBATCH --error=monte-carlo_%j.err
#SBATCH --time=00:30:00
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=8G

# Load Apptainer
module load apptainer

# Define paths
CONTAINER=~/containers/monte-carlo.sif
OUTPUT_DIR=/scratch/$USER/mc_results_$SLURM_JOB_ID

# Create output directory
mkdir -p $OUTPUT_DIR

echo "Job $SLURM_JOB_ID started on $(hostname) at $(date)"
echo "Container: $CONTAINER"
echo "Output:    $OUTPUT_DIR"

# Run the simulation
apptainer exec \
    --bind $OUTPUT_DIR:/output \
    $CONTAINER python /app/estimate_pi.py 50000000 \
        --trials 20 \
        --output /output/results.json

echo "Job completed at $(date)"
echo "Results:"
cat $OUTPUT_DIR/results.json
```

Let's walk through the key parts:

**`#SBATCH` directives** tell SLURM what resources to allocate. `%j` in the output/error filenames is replaced with the job ID, so each run produces uniquely named log files.

**`module load apptainer`** makes the `apptainer` command available. Always include this in your job script — modules loaded in your interactive session aren't inherited by batch jobs.

**`mkdir -p $OUTPUT_DIR`** creates a job-specific output directory on scratch. Using `$SLURM_JOB_ID` in the path keeps results from different runs separate.

**`--bind $OUTPUT_DIR:/output`** mounts the scratch directory as `/output` inside the container. The Python script writes to `/output/results.json`, which actually lands on scratch.

Submit the job:

```bash
sbatch monte-carlo.slurm
```

Monitor it:

```bash
# Check job status
squeue -u $USER

# Once completed, view the output
cat monte-carlo_*.out
cat /scratch/$USER/mc_results_*/results.json
```

<!-- TODO: Add a screenshot showing squeue output with the container job running -->

## MPI Containers

MPI (Message Passing Interface) enables running a single program across multiple nodes, with processes communicating over the high-speed network. MPI containers require special handling because the MPI library inside the container must be compatible with the host system's MPI and network drivers.

### The Concept

The recommended approach is the **hybrid model**: the container includes MPI libraries, but the *host* MPI launcher (`mpirun` or the cluster's equivalent) starts the processes. This lets the host handle network configuration and process placement while the container provides the application environment.

```
  Host System                    Container
┌─────────────────┐          ┌──────────────────┐
│  mpirun / ibrun │ launches │  Your app + MPI   │
│  (host MPI)     │ ───────> │  (container MPI)  │
│                 │          │                    │
│  Manages:       │          │  Provides:         │
│  - Network      │          │  - Application     │
│  - Process      │          │  - Dependencies    │
│    placement    │          │  - Compatible MPI  │
└─────────────────┘          └──────────────────┘
```

**Important:** The MPI version inside the container should match or be newer than the host's MPI version. Using the same major version of OpenMPI generally works.

### The Example: Parallel Pi Estimation

We'll create a version of the Monte Carlo simulation that distributes work across multiple MPI processes. Each process estimates Pi with a portion of the total samples, and the results are combined.

On your workstation, create the project:

```bash
mkdir ~/monte-carlo-mpi && cd ~/monte-carlo-mpi
```

Create `estimate_pi_mpi.py`:

```python
#!/usr/bin/env python3
"""Parallel Monte Carlo Pi estimation using MPI."""

from mpi4py import MPI
import numpy as np
import time
import socket
import argparse

def monte_carlo_pi(num_samples):
    x = np.random.uniform(0, 1, num_samples)
    y = np.random.uniform(0, 1, num_samples)
    inside = np.sum(x**2 + y**2 <= 1)
    return inside

def main():
    comm = MPI.COMM_WORLD
    rank = comm.Get_rank()
    size = comm.Get_size()

    parser = argparse.ArgumentParser()
    parser.add_argument("samples", type=int, nargs="?", default=10_000_000)
    args = parser.parse_args()

    # Each process handles an equal share of the samples
    samples_per_rank = args.samples // size
    np.random.seed(rank * 1000 + 42)

    if rank == 0:
        print(f"Estimating Pi with {args.samples:,} total samples across {size} processes")
        start = time.time()

    # Each rank computes its portion
    local_inside = monte_carlo_pi(samples_per_rank)

    # Combine results across all ranks
    total_inside = comm.reduce(local_inside, op=MPI.SUM, root=0)

    if rank == 0:
        pi_estimate = 4 * total_inside / (samples_per_rank * size)
        elapsed = time.time() - start
        print(f"  Estimate: {pi_estimate:.8f}")
        print(f"  Error:    {abs(pi_estimate - np.pi):.8f}")
        print(f"  Time:     {elapsed:.3f}s")
        print(f"  Speedup:  {size} processes")

    # Every rank reports its hostname for verification
    print(f"  Rank {rank}/{size} on {socket.gethostname()}: {local_inside:,} hits "
          f"from {samples_per_rank:,} samples")

if __name__ == "__main__":
    main()
```

Create the `Dockerfile`:

```dockerfile
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        python3 \
        python3-pip \
        openmpi-bin \
        libopenmpi-dev \
    && rm -rf /var/lib/apt/lists/*

RUN pip3 install --no-cache-dir mpi4py numpy

WORKDIR /app
COPY estimate_pi_mpi.py .
RUN chmod +x estimate_pi_mpi.py

CMD ["python3", "estimate_pi_mpi.py"]
```

Build and push:

```bash
docker build -t yourusername/monte-carlo-mpi:0.1 .
docker push yourusername/monte-carlo-mpi:0.1
```

Test locally with Docker before using cluster :

```bash
# Single process
docker run --rm yourusername/monte-carlo-mpi:0.1 \
    mpirun -n 1 python3 /app/estimate_pi_mpi.py 5000000

# Two processes (if Docker has access to multiple cores)
docker run --rm yourusername/monte-carlo-mpi:0.1 \
    mpirun -n 2 python3 /app/estimate_pi_mpi.py 5000000
```

On ACE HPC, pull the image and create a batch script called `mpi-pi.slurm`:

```bash
#!/bin/bash
#SBATCH --job-name=mpi-pi
#SBATCH --output=mpi-pi_%j.out
#SBATCH --error=mpi-pi_%j.err
#SBATCH --time=00:30:00
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=8
#SBATCH --mem=16G

module load apptainer
module load openmpi

CONTAINER=~/containers/monte-carlo-mpi.sif

echo "MPI Pi estimation: $SLURM_NTASKS total tasks across $SLURM_NNODES nodes"

# Use the HOST mpirun to launch container processes.
# This ensures proper network configuration and process placement.
mpirun -np $SLURM_NTASKS apptainer exec $CONTAINER \
    python3 /app/estimate_pi_mpi.py 100000000
```

Submit:

```bash
sbatch mpi-pi.slurm
```

You should see each rank report its hostname — ranks on the same node share a hostname, ranks on different nodes report different hostnames. This confirms MPI is communicating across nodes.

## GPU Containers

GPU containers let you run CUDA workloads (deep learning, molecular dynamics, etc.) inside a container. Apptainer exposes the host's NVIDIA GPUs to the container with the `--nv` flag.

### The Concept

The `--nv` flag tells Apptainer to bind-mount the host's NVIDIA driver libraries and GPU devices into the container. The container must include CUDA libraries that are compatible with the host's NVIDIA driver version. As a rule: the host driver must be **equal to or newer** than the CUDA version in the container.

You don't need to install NVIDIA drivers inside the container — `--nv` injects them from the host at runtime.

### The Example: GPU Matrix Benchmark

We'll create a container that benchmarks GPU performance with a matrix multiplication using PyTorch, then compare it to CPU performance.

On your workstation, create `gpu_benchmark.py`:

```python
#!/usr/bin/env python3
"""Benchmark matrix multiplication on CPU vs GPU."""

import torch
import time
import argparse
import os

def benchmark(device, size, iterations):
    """Run matrix multiplication benchmark on the given device."""
    a = torch.randn(size, size, device=device)
    b = torch.randn(size, size, device=device)

    # Warm-up
    torch.matmul(a, b)
    if device.type == "cuda":
        torch.cuda.synchronize()

    start = time.time()
    for _ in range(iterations):
        torch.matmul(a, b)
    if device.type == "cuda":
        torch.cuda.synchronize()
    elapsed = time.time() - start

    # FLOPS for matrix multiplication: 2 * N^3 per multiply
    flops = 2 * size**3 * iterations / elapsed
    return elapsed, flops

def main():
    parser = argparse.ArgumentParser(description="GPU matrix multiplication benchmark")
    parser.add_argument("--size", type=int, default=4096, help="Matrix size NxN")
    parser.add_argument("--iterations", type=int, default=10)
    parser.add_argument("--no-gpu", action="store_true", help="Skip GPU benchmark")
    args = parser.parse_args()

    job_id = os.environ.get("SLURM_JOB_ID", "local")
    print(f"Matrix Benchmark (Job: {job_id})")
    print(f"  PyTorch: {torch.__version__}")
    print(f"  Matrix size: {args.size}x{args.size}")
    print(f"  Iterations: {args.iterations}")
    print()

    # CPU benchmark
    cpu_time, cpu_flops = benchmark(torch.device("cpu"), args.size, args.iterations)
    print(f"  CPU: {cpu_time:.3f}s ({cpu_flops/1e9:.1f} GFLOPS)")

    # GPU benchmark
    if not args.no_gpu and torch.cuda.is_available():
        print(f"  CUDA: {torch.version.cuda}")
        for i in range(torch.cuda.device_count()):
            props = torch.cuda.get_device_properties(i)
            print(f"  GPU {i}: {props.name} ({props.total_memory / 1e9:.1f} GB)")

        gpu_time, gpu_flops = benchmark(torch.device("cuda"), args.size, args.iterations)
        print(f"  GPU: {gpu_time:.3f}s ({gpu_flops/1e12:.2f} TFLOPS)")
        print(f"  Speedup: {cpu_time/gpu_time:.1f}x")
    elif args.no_gpu:
        print("  GPU: skipped (--no-gpu)")
    else:
        print("  GPU: not available. Did you use the --nv flag?")

if __name__ == "__main__":
    main()
```

Create the `Dockerfile`:

```dockerfile
FROM nvidia/cuda:12.2.0-cudnn8-runtime-ubuntu22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        python3 \
        python3-pip \
    && rm -rf /var/lib/apt/lists/*

RUN pip3 install --no-cache-dir \
    torch torchvision torchaudio \
    --index-url https://download.pytorch.org/whl/cu121

WORKDIR /app
COPY gpu_benchmark.py .

CMD ["python3", "gpu_benchmark.py"]
```

Build and push:

```bash
docker build -t yourusername/gpu-benchmark:0.1 .
docker push yourusername/gpu-benchmark:0.1
```

On ACE HPC, pull the image and create `gpu-bench.slurm`:

```bash
#!/bin/bash
#SBATCH --job-name=gpu-bench
#SBATCH --output=gpu-bench_%j.out
#SBATCH --error=gpu-bench_%j.err
#SBATCH --time=00:15:00
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --gres=gpu:1
#SBATCH --partition=gpu

module load apptainer

CONTAINER=~/containers/gpu-benchmark.sif

echo "GPU Benchmark - Job $SLURM_JOB_ID on $(hostname)"

# The --nv flag exposes the host GPU to the container
apptainer exec --nv $CONTAINER python3 /app/gpu_benchmark.py --size 8192 --iterations 20
```

Submit:

```bash
sbatch gpu-bench.slurm
```

The output will show CPU GFLOPS vs GPU TFLOPS — typically a 30–100x speedup for matrix operations, depending on the GPU model and matrix size.

Without `--nv`, the container cannot see the GPU and `torch.cuda.is_available()` returns `False`. This is the most common mistake when running GPU containers.

<!-- TODO: Add a screenshot showing the GPU benchmark output comparing CPU vs GPU performance -->

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| `command not found` inside container | Binary isn't in the container's `$PATH` | Check with `apptainer exec container.sif which python` or use the full path |
| GPU not detected | Missing `--nv` flag | Add `--nv` to the `apptainer exec` command |
| GPU not detected | No GPU allocated by SLURM | Add `#SBATCH --gres=gpu:1` and `#SBATCH --partition=gpu` |
| Permission denied on `.sif` file | File permissions | `chmod +r container.sif` |
| Out of disk space during pull | Cache fills home directory | Set `APPTAINER_CACHEDIR=/scratch/$USER/.apptainer` |
| MPI processes all on same node | Not using host MPI launcher | Use `mpirun -np $SLURM_NTASKS apptainer exec ...` not `apptainer exec ... mpirun` |
| Host environment leaking in | Host `$PATH` or Python paths interfere | Use `--cleanenv` to start with a clean environment |

## Quick Reference

```bash
# Load the module
module load apptainer

# Pull a Docker image
apptainer pull docker://image:tag
apptainer pull custom_name.sif docker://image:tag

# Run a command in a container
apptainer exec image.sif command args

# Interactive shell
apptainer shell image.sif

# Run with GPU
apptainer exec --nv image.sif command

# Run with custom bind mounts
apptainer exec --bind /host/path:/container/path:ro image.sif command

# Run with clean environment
apptainer exec --cleanenv image.sif command

# Pass environment variables
apptainer exec --env KEY=VALUE image.sif command

# Inspect image metadata
apptainer inspect image.sif

# Manage cache
apptainer cache list
apptainer cache clean
```

---

**See also:** [SLURM Basics](../running-jobs/slurm-basics) | [Writing Job Scripts](../running-jobs/job-scripts) | [Apptainer User Guide](https://apptainer.org/docs/user/latest/)

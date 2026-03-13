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
$ salloc --ntasks=1 --mem=4G --time=01:00:00

# Load Apptainer
$ module load apptainer

# Verify
$ apptainer --version
```

### Pulling an Image

Apptainer can pull any Docker image and convert it to its native `.sif` format:

```bash
# Pull the Monte Carlo image we built in previous tutorials
$ apptainer pull docker://yourusername/monte-carlo:0.1

# This creates: monte-carlo_0.1.sif
$ ls -lh monte-carlo_0.1.sif
```

The `.sif` file is a single, portable file containing the entire container. You can copy it, move it, share it — it's just a file.

To pull to a specific directory or with a custom filename:

```bash
$ mkdir -p ~/containers
$ apptainer pull ~/containers/monte-carlo.sif docker://yourusername/monte-carlo:0.1
```

:::note Cache management
Apptainer caches downloaded layers in `~/.apptainer/cache`. This counts toward your `$HOME` quota. Clean the cache regularly if you are pulling many images:
```bash
apptainer cache clean
```
You can check cache size with `apptainer cache list`.
:::

### Running Containers

:::danger Do not run containers on the head node
The commands below are for reference only. **Never run `apptainer exec`, `apptainer run`, or `apptainer shell` on the head node for actual compute work.** The head node is a shared login environment — running workloads there degrades it for all users and may result in your session being terminated.

All container execution must be done inside a SLURM job. Use `sbatch` for batch jobs or `salloc` for interactive sessions on a compute node. See the [SLURM Batch Jobs](#slurm-batch-jobs) section below for the correct approach.
:::

Apptainer has three commands for running containers:

**`apptainer exec`** — Run a specific command inside the container. This is what you'll use most often:

```bash
# Run the Monte Carlo simulation
$ apptainer exec monte-carlo_0.1.sif python /app/estimate_pi.py 5000000

# Check what Python version is inside the container
$ apptainer exec monte-carlo_0.1.sif python --version

# List installed packages
$ apptainer exec monte-carlo_0.1.sif pip list
```

**`apptainer shell`** — Start an interactive shell inside the container for exploration and debugging:

```bash
$ apptainer shell monte-carlo_0.1.sif
Apptainer> python --version
Apptainer> ls /app/
Apptainer> cat /etc/os-release
Apptainer> exit
```

Notice that inside the shell, you're still *you* — your home directory is accessible, your files are there, and you have the same permissions as outside. This is different from Docker, where you'd typically be root.

**`apptainer run`** — Execute the container's default command (its `CMD` or `ENTRYPOINT`):

```bash
$ apptainer run monte-carlo_0.1.sif
```


### Interactive Shell Sessions with srun

When you need to explore a container interactively — inspect its filesystem, test commands, or debug a failing job — use `srun` to get an interactive session on a compute node first, then launch `apptainer shell` from there.

```bash
# Request an interactive session on a compute node
$ srun --ntasks=1 --cpus-per-task=2 --mem=4G --time=01:00:00 --pty bash
```

Once your shell lands on a compute node, load Apptainer and open the container:

```bash
$ module load apptainer
$ apptainer shell ~/containers/monte-carlo.sif
```

You'll drop into the container's environment:

```
Apptainer> python --version
Python 3.11.x
Apptainer> pip list
Apptainer> ls /app/
Apptainer> python /app/estimate_pi.py 100000
Apptainer> exit
```

If you need a GPU node for interactive debugging:

```bash
$ srun --ntasks=1 --cpus-per-task=4 --mem=16G --time=01:00:00 \
     --gres=gpu:1 --partition=gpu --pty bash

# Then inside the compute node:
$ module load apptainer
$ apptainer shell --nv ~/containers/gpu-benchmark.sif
```

:::tip
`apptainer shell` inherits your home directory and current working directory automatically, so your data and scripts are accessible inside the container without any extra `--bind` flags.
:::

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
OUTPUT_DIR=mc_results_$SLURM_JOB_ID

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

**`mkdir -p $OUTPUT_DIR`** creates a job-specific output directory. Using `$SLURM_JOB_ID` in the path keeps results from different runs separate.

**`--bind $OUTPUT_DIR:/output`** mounts the scratch directory as `/output` inside the container. The Python script writes to `/output/results.json`, which actually lands on scratch.

Submit the job:

```bash
$ sbatch monte-carlo.slurm
```

Monitor it:

```bash
# Check job status
$ squeue -u $USER

# Once completed, view the output
$ cat monte-carlo_*.out
$ cat mc_results_*/results.json
```

<!-- TODO: Add a screenshot showing squeue output with the container job running -->

## MPI Containers

MPI (Message Passing Interface) enables running a single program across multiple nodes, with processes communicating over the high-speed network. MPI containers require special handling because the MPI library inside the container must be compatible with the host system's MPI and network drivers.

### The Concept

The recommended approach is the **hybrid model**: the container includes MPI libraries, but the *host* MPI launcher (`mpirun` or the cluster's equivalent) starts the processes. This lets the host handle network configuration and process placement while the container provides the application environment.

```
  Host System                    Container
┌─────────────────┐          ┌──────────────────┐
│  mpirun / ibrun │ launches │  Your app + MPI  │
│  (host MPI)     │ ───────> │  (container MPI) │
│                 │          │                  │
│  Manages:       │          │  Provides:       │
│  - Network      │          │  - Application   │
│  - Process      │          │  - Dependencies  │
│    placement    │          │  - Compatible MPI│
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
$ docker build --platform linux/amd64 -t yourusername/monte-carlo-mpi:0.1 .
[+] Building 51.6s (12/12) FINISHED                                                                                                                                                            docker:desktop-linux
 => [internal] load build definition from Dockerfile                                                                                                                                                           0.0s
 => => transferring dockerfile: 446B                                                                                                                                                                           0.0s
 => [internal] load metadata for docker.io/library/ubuntu:22.04                                                                                                                                                1.0s
 => [auth] library/ubuntu:pull token for registry-1.docker.io                                                                                                                                                  0.0s
 => [internal] load .dockerignore                                                                                                                                                                              0.0s
 => => transferring context: 2B                                                                                                                                                                                0.0s
 => [1/6] FROM docker.io/library/ubuntu:22.04@sha256:3ba65aa20f86a0fad9df2b2c259c613df006b2e6d0bfcc8a146afb8c525a9751                                                                                          1.1s
 => => resolve docker.io/library/ubuntu:22.04@sha256:3ba65aa20f86a0fad9df2b2c259c613df006b2e6d0bfcc8a146afb8c525a9751                                                                                          0.0s
 => => sha256:b1cba2e842ca52b95817f958faf99734080c78e92e43ce609cde9244867b49ed 29.54MB / 29.54MB                                                                                                               0.8s
 => => extracting sha256:b1cba2e842ca52b95817f958faf99734080c78e92e43ce609cde9244867b49ed                                                                                                                      0.3s
 => [internal] load build context                                                                                                                                                                              0.0s
 => => transferring context: 1.54kB                                                                                                                                                                            0.0s
 => [2/6] RUN apt-get update     && apt-get install -y --no-install-recommends         python3         python3-pip         openmpi-bin         libopenmpi-dev     && rm -rf /var/lib/apt/lists/*              40.4s
 => [3/6] RUN pip3 install --no-cache-dir mpi4py numpy                                                                                                                                                         2.5s
 => [4/6] WORKDIR /app                                                                                                                                                                                         0.0s
 => [5/6] COPY estimate_pi_mpi.py .                                                                                                                                                                            0.0s
 => [6/6] RUN chmod +x estimate_pi_mpi.py                                                                                                                                                                      0.1s
 => exporting to image                                                                                                                                                                                         6.3s
 => => exporting layers                                                                                                                                                                                        6.3s
 => => exporting manifest sha256:6ef2c750407397b709e1b0d616052e9fc6626b5a221b03302676f83ff1d2b6f0                                                                                                              0.0s
 => => exporting config sha256:99c2625718bb3a900f9e2efe51abf1c10f9c6e4b5b25806b543864a2feb55fcc                                                                                                                0.0s
 => => exporting attestation manifest sha256:5562fc2c080dc34fe0944bca9d807260209078cafb8b83021c9cddea0a9c157a                                                                                                  0.0s
 => => exporting manifest list sha256:2b9a09f20de16bc9187464b0cc04c7c33db9da1239d9991588ddf2395030bb86                                                                                                         0.0s
 => => naming to docker.io/ianwasukira/monte-carlo-mpi:0.1                                                                                                                                                     0.0s
```

```
docker push yourusername/monte-carlo-mpi:0.1
The push refers to repository [docker.io/yourusername/monte-carlo-mpi]
54c6a19e1ba5: Pushed
195d96c8233d: Pushed
7125307ec0a6: Pushed
64861010b1bc: Pushed
16a2ff661819: Pushed
b1cba2e842ca: Pushed
658d7018384c: Pushed
0.1: digest: sha256:2b9a09f20de16bc9187464b0cc04c7c33db9da1239d9991588ddf2395030bb86 size: 856
```

Test locally with Docker before using cluster :

```bash
# Single process
$ docker run --rm yourusername/monte-carlo-mpi:0.1 \
    mpirun -n 1 python3 /app/estimate_pi_mpi.py 5000000

# Two processes (if Docker has access to multiple cores)
$ docker run --rm yourusername/monte-carlo-mpi:0.1 \
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
$ sbatch mpi_pi.slurm
Submitted batch job 1197
$ cat mpi-pi_1197.out
MPI Pi estimation: 8 total tasks across 1 nodes
Estimating Pi with 100,000,000 total samples across 8 processes
  Rank 5/8 on kla-ac-cpu-45: 9,818,024 hits from 12,500,000 samples
  Rank 6/8 on kla-ac-cpu-45: 9,813,591 hits from 12,500,000 samples
  Rank 7/8 on kla-ac-cpu-45: 9,814,575 hits from 12,500,000 samples
  Rank 1/8 on kla-ac-cpu-45: 9,817,978 hits from 12,500,000 samples
  Rank 2/8 on kla-ac-cpu-45: 9,818,049 hits from 12,500,000 samples
  Rank 3/8 on kla-ac-cpu-45: 9,818,775 hits from 12,500,000 samples
  Rank 4/8 on kla-ac-cpu-45: 9,817,634 hits from 12,500,000 samples
  Estimate: 3.14142368
  Error:    0.00016897
  Time:     2.563s
  Speedup:  8 processes
  Rank 0/8 on kla-ac-cpu-45: 9,816,966 hits from 12,500,000 samples
```

See each rank report its hostname — ranks on the same node share a hostname, ranks on different nodes report different hostnames. This confirms MPI is communicating across nodes.

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
$ docker build -t yourusername/gpu-benchmark:0.1 .
$ docker push yourusername/gpu-benchmark:0.1
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
$ sbatch gpu-bench.slurm
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

**Next:** [Advanced Build Topics](advanced-builds) — reduce image size with multi-stage builds and support multiple CPU architectures.

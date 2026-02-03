---
id: advanced-builds
title: Advanced Build Topics
sidebar_label: Advanced Build Topics
sidebar_position: 3
---

# Advanced Build Topics

This tutorial covers advanced container building techniques including multi-stage builds, GPU-enabled containers, MPI support, and optimization strategies for HPC workloads.

## Learning Objectives

After completing this tutorial, you will be able to:

- Use multi-stage builds to create smaller, more efficient images
- Build GPU-enabled containers for CUDA and machine learning
- Create MPI-enabled containers for parallel computing
- Optimize containers for size and performance
- Implement security best practices

---

## Multi-Stage Builds

Multi-stage builds allow you to use multiple `FROM` statements in a single Dockerfile. This is powerful for separating build-time dependencies from runtime dependencies, resulting in much smaller final images.

### The Problem: Large Images

A typical build includes compilers, development headers, and build tools:

```dockerfile
# Single-stage build (large image)
FROM python:3.11

# Install build tools
RUN apt-get update && apt-get install -y \
    build-essential \
    gfortran \
    libopenblas-dev

# Install Python packages (some require compilation)
RUN pip install numpy scipy pandas scikit-learn

# Your application
COPY app.py /app/
CMD ["python", "/app/app.py"]

# Result: ~2GB image with unnecessary build tools
```

### The Solution: Multi-Stage Build

```dockerfile
# Stage 1: Builder
FROM python:3.11 AS builder

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    gfortran \
    libopenblas-dev

# Create virtual environment and install packages
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
RUN pip install --no-cache-dir numpy scipy pandas scikit-learn

# Stage 2: Runtime (final image)
FROM python:3.11-slim

# Install only runtime dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    libopenblas0 \
    libgomp1 \
    && rm -rf /var/lib/apt/lists/*

# Copy the virtual environment from builder
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Your application
WORKDIR /app
COPY app.py .
CMD ["python", "app.py"]

# Result: ~500MB image with only what's needed
```

### Exercise 1: Multi-Stage Build for C++ Application

```bash
mkdir -p ~/multistage-demo
cd ~/multistage-demo
```

Create a simple C++ program:

```bash
cat > hello.cpp << 'EOF'
#include <iostream>
#include <string>

int main(int argc, char* argv[]) {
    std::string name = (argc > 1) ? argv[1] : "World";
    std::cout << "Hello, " << name << "!" << std::endl;
    std::cout << "This binary was compiled inside a container." << std::endl;
    return 0;
}
EOF
```

Create a multi-stage Dockerfile:

```bash
cat > Dockerfile << 'EOF'
# Stage 1: Build
FROM gcc:12 AS builder

WORKDIR /build
COPY hello.cpp .
RUN g++ -static -o hello hello.cpp

# Stage 2: Runtime
FROM debian:bookworm-slim

WORKDIR /app
COPY --from=builder /build/hello .

ENTRYPOINT ["./hello"]
EOF
```

Build and compare sizes:

```bash
# Build multi-stage version
docker build -t hello-multi:1.0 .

# Build single-stage for comparison
cat > Dockerfile.single << 'EOF'
FROM gcc:12
WORKDIR /app
COPY hello.cpp .
RUN g++ -static -o hello hello.cpp
ENTRYPOINT ["./hello"]
EOF

docker build -f Dockerfile.single -t hello-single:1.0 .

# Compare sizes
docker images | grep hello
```

**Expected output:**
```
hello-single   1.0    ...    1.4GB
hello-multi    1.0    ...    80MB
```

---

## GPU-Enabled Containers

### NVIDIA CUDA Containers

For GPU computing, start with NVIDIA's official CUDA images:

```dockerfile
# PyTorch with CUDA
FROM nvidia/cuda:12.2.0-cudnn8-runtime-ubuntu22.04

# Install Python
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

# Install PyTorch with CUDA support
RUN pip3 install --no-cache-dir \
    torch \
    torchvision \
    torchaudio \
    --index-url https://download.pytorch.org/whl/cu121

WORKDIR /app
CMD ["python3"]
```

### CUDA Image Variants

| Image Tag | Contents | Size | Use Case |
|-----------|----------|------|----------|
| `nvidia/cuda:12.2.0-base-ubuntu22.04` | CUDA runtime only | ~200MB | Running pre-compiled CUDA apps |
| `nvidia/cuda:12.2.0-runtime-ubuntu22.04` | Runtime + cuBLAS, cuFFT | ~1.5GB | Most ML inference |
| `nvidia/cuda:12.2.0-devel-ubuntu22.04` | Runtime + compilers | ~4GB | Compiling CUDA code |
| `nvidia/cuda:12.2.0-cudnn8-runtime-ubuntu22.04` | Runtime + cuDNN | ~2GB | Deep learning |

### Exercise 2: PyTorch GPU Container

```bash
mkdir -p ~/pytorch-gpu
cd ~/pytorch-gpu
```

Create a GPU test script:

```bash
cat > test_gpu.py << 'EOF'
#!/usr/bin/env python3
"""Test GPU availability and run a simple computation."""

import torch
import time

def main():
    print("=" * 60)
    print("PyTorch GPU Test")
    print("=" * 60)

    print(f"\nPyTorch version: {torch.__version__}")
    print(f"CUDA available: {torch.cuda.is_available()}")

    if torch.cuda.is_available():
        print(f"CUDA version: {torch.version.cuda}")
        print(f"cuDNN version: {torch.backends.cudnn.version()}")
        print(f"Number of GPUs: {torch.cuda.device_count()}")

        for i in range(torch.cuda.device_count()):
            props = torch.cuda.get_device_properties(i)
            print(f"\nGPU {i}: {props.name}")
            print(f"  Memory: {props.total_memory / 1024**3:.1f} GB")
            print(f"  Compute capability: {props.major}.{props.minor}")

        # Benchmark: Matrix multiplication
        print("\n" + "=" * 60)
        print("Running GPU benchmark...")
        size = 10000

        # Create random matrices on GPU
        a = torch.randn(size, size, device='cuda')
        b = torch.randn(size, size, device='cuda')

        # Warm-up
        torch.matmul(a, b)
        torch.cuda.synchronize()

        # Benchmark
        start = time.time()
        for _ in range(10):
            c = torch.matmul(a, b)
        torch.cuda.synchronize()
        elapsed = time.time() - start

        print(f"10x Matrix multiplication ({size}x{size}): {elapsed:.3f} seconds")
        print(f"Throughput: {10 * 2 * size**3 / elapsed / 1e12:.2f} TFLOPS")
    else:
        print("\nNo GPU available. Running on CPU.")
        print("To enable GPU, run with: apptainer exec --nv container.sif ...")

    print("=" * 60)

if __name__ == "__main__":
    main()
EOF
```

Create the Dockerfile:

```bash
cat > Dockerfile << 'EOF'
FROM nvidia/cuda:12.2.0-cudnn8-runtime-ubuntu22.04

LABEL maintainer="your.email@example.com"
LABEL description="PyTorch with CUDA support"

# Prevent interactive prompts
ENV DEBIAN_FRONTEND=noninteractive

# Install Python and dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

# Install PyTorch with CUDA
RUN pip3 install --no-cache-dir \
    torch \
    torchvision \
    torchaudio \
    --index-url https://download.pytorch.org/whl/cu121

WORKDIR /app
COPY test_gpu.py .

CMD ["python3", "test_gpu.py"]
EOF
```

Build and push:

```bash
docker build -t pytorch-gpu:1.0 .
docker tag pytorch-gpu:1.0 yourusername/pytorch-gpu:1.0
docker push yourusername/pytorch-gpu:1.0
```

On ACE HPC:
```bash
module load apptainer
apptainer exec --nv docker://yourusername/pytorch-gpu:1.0 python3 /app/test_gpu.py
```

### TensorFlow GPU Container

```dockerfile
# TensorFlow with GPU
FROM tensorflow/tensorflow:2.14.0-gpu

LABEL description="TensorFlow with GPU support"

# Install additional packages
RUN pip install --no-cache-dir \
    pandas \
    matplotlib \
    scikit-learn

WORKDIR /app
COPY train.py .

CMD ["python", "train.py"]
```

---

## MPI Containers for Parallel Computing

Message Passing Interface (MPI) enables distributed parallel computing across multiple nodes.

### MPI Container Strategy

The key challenge with MPI containers is that the MPI version inside the container should be compatible with the host system's MPI. There are two approaches:

1. **Hybrid approach** (recommended): Use host MPI to launch container processes
2. **Container MPI**: Include MPI in the container (simpler but may have compatibility issues)

### Exercise 3: MPI Container

```bash
mkdir -p ~/mpi-container
cd ~/mpi-container
```

Create an MPI test program:

```bash
cat > mpi_hello.py << 'EOF'
#!/usr/bin/env python3
"""Simple MPI hello world with mpi4py."""

from mpi4py import MPI
import socket

def main():
    comm = MPI.COMM_WORLD
    rank = comm.Get_rank()
    size = comm.Get_size()
    hostname = socket.gethostname()

    print(f"Hello from rank {rank}/{size} on {hostname}")

    # Simple collective operation
    comm.Barrier()

    if rank == 0:
        print(f"\n{'='*50}")
        print(f"MPI Info:")
        print(f"  Total processes: {size}")
        print(f"  MPI Version: {MPI.Get_version()}")
        print(f"{'='*50}")

if __name__ == "__main__":
    main()
EOF
```

Create the Dockerfile:

```bash
cat > Dockerfile << 'EOF'
FROM ubuntu:22.04

LABEL description="MPI-enabled Python container"

ENV DEBIAN_FRONTEND=noninteractive

# Install MPI and Python
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 \
    python3-pip \
    openmpi-bin \
    libopenmpi-dev \
    && rm -rf /var/lib/apt/lists/*

# Install mpi4py
RUN pip3 install --no-cache-dir mpi4py numpy

WORKDIR /app
COPY mpi_hello.py .

# Default to running the MPI program
CMD ["python3", "mpi_hello.py"]
EOF
```

Build:

```bash
docker build -t mpi-hello:1.0 .
```

### Running MPI Containers on ACE HPC

```bash
#!/bin/bash
#SBATCH --job-name=mpi-container
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=4
#SBATCH --time=00:30:00

module load apptainer
module load openmpi

# Run MPI across nodes using host MPI
mpirun -np $SLURM_NTASKS apptainer exec mpi-hello.sif python3 /app/mpi_hello.py
```

---

## Optimization Techniques

### 1. Minimize Layers

```dockerfile
# Bad: Many layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
RUN apt-get clean

# Good: Single layer
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        curl \
        git \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

### 2. Order by Change Frequency

```dockerfile
# Things that rarely change first
FROM python:3.11-slim

# System dependencies (rarely change)
RUN apt-get update && apt-get install -y --no-install-recommends \
    libgomp1 \
    && rm -rf /var/lib/apt/lists/*

# Python dependencies (change occasionally)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Application code (changes frequently)
COPY src/ /app/src/
```

### 3. Use .dockerignore

```bash
# .dockerignore
.git
.gitignore
__pycache__
*.pyc
.pytest_cache
.venv
venv
*.egg-info
dist
build
.tox
.coverage
htmlcov
*.log
.env
.env.*
Dockerfile
docker-compose*.yml
*.md
!README.md
data/
output/
```

### 4. Pin Versions for Reproducibility

```dockerfile
# requirements.txt with pinned versions
FROM python:3.11.4-slim

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
```

```txt
# requirements.txt
numpy==1.24.3
pandas==2.0.3
scikit-learn==1.3.0
matplotlib==3.7.2
```

### 5. Use Specific Base Image Digests

```dockerfile
# Most reproducible: use digest
FROM python:3.11.4-slim@sha256:abc123...

# Good: specific version
FROM python:3.11.4-slim

# Avoid: tag can change
FROM python:3.11-slim
FROM python:latest
```

---

## Security Best Practices

### 1. Run as Non-Root User

```dockerfile
FROM python:3.11-slim

# Create non-root user
RUN useradd -m -s /bin/bash appuser

WORKDIR /app
COPY --chown=appuser:appuser . .

# Switch to non-root user
USER appuser

CMD ["python", "app.py"]
```

### 2. Never Include Secrets

```dockerfile
# NEVER do this
ENV API_KEY=secret123
COPY credentials.json /app/

# Instead, mount at runtime:
# docker run -v ~/.config/myapp:/config:ro myimage
# apptainer exec --bind ~/.config/myapp:/config:ro container.sif
```

### 3. Use Official Base Images

```dockerfile
# Good: Official images
FROM python:3.11-slim
FROM ubuntu:22.04
FROM nvidia/cuda:12.2.0-runtime-ubuntu22.04

# Avoid: Unknown sources
FROM random-user/mystery-image:latest
```

### 4. Verify Downloads

```dockerfile
RUN curl -fsSL https://example.com/file.tar.gz -o file.tar.gz \
    && echo "abc123... file.tar.gz" | sha256sum -c - \
    && tar xzf file.tar.gz \
    && rm file.tar.gz
```

### 5. Minimize Attack Surface

```dockerfile
# Remove unnecessary packages
RUN apt-get update \
    && apt-get install -y --no-install-recommends needed-package \
    && apt-get purge -y --auto-remove \
    && rm -rf /var/lib/apt/lists/*

# Don't install documentation
RUN pip install --no-cache-dir --no-compile package
```

---

## Exercise 4: Optimized ML Container

Let's build a production-ready ML container using all techniques:

```bash
mkdir -p ~/optimized-ml
cd ~/optimized-ml
```

Create requirements:

```bash
cat > requirements.txt << 'EOF'
numpy==1.24.3
pandas==2.0.3
scikit-learn==1.3.0
joblib==1.3.2
EOF
```

Create the application:

```bash
cat > train_model.py << 'EOF'
#!/usr/bin/env python3
"""Train a simple ML model."""

import argparse
import joblib
import numpy as np
import pandas as pd
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import classification_report

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--output', '-o', default='/output/model.joblib')
    parser.add_argument('--n-estimators', type=int, default=100)
    args = parser.parse_args()

    print("Loading data...")
    iris = load_iris()
    X_train, X_test, y_train, y_test = train_test_split(
        iris.data, iris.target, test_size=0.2, random_state=42
    )

    print(f"Training RandomForest with {args.n_estimators} estimators...")
    model = RandomForestClassifier(n_estimators=args.n_estimators, random_state=42)
    model.fit(X_train, y_train)

    # Evaluate
    cv_scores = cross_val_score(model, X_train, y_train, cv=5)
    print(f"Cross-validation accuracy: {cv_scores.mean():.4f} (+/- {cv_scores.std()*2:.4f})")

    y_pred = model.predict(X_test)
    print("\nTest set classification report:")
    print(classification_report(y_test, y_pred, target_names=iris.target_names))

    # Save model
    joblib.dump(model, args.output)
    print(f"\nModel saved to {args.output}")

if __name__ == "__main__":
    main()
EOF
```

Create the optimized Dockerfile:

```bash
cat > Dockerfile << 'EOF'
# Stage 1: Builder
FROM python:3.11-slim AS builder

WORKDIR /build

# Install build dependencies
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
    && rm -rf /var/lib/apt/lists/*

# Create venv and install packages
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt


# Stage 2: Runtime
FROM python:3.11-slim

# Metadata
LABEL maintainer="your.email@example.com"
LABEL description="Optimized ML training container"
LABEL version="1.0"

# Install only runtime dependencies
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        libgomp1 \
    && rm -rf /var/lib/apt/lists/* \
    && useradd -m -s /bin/bash mluser

# Copy virtual environment
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Set up application
WORKDIR /app
COPY --chown=mluser:mluser train_model.py .

# Create output directory
RUN mkdir -p /output && chown mluser:mluser /output

# Switch to non-root user
USER mluser

ENTRYPOINT ["python", "train_model.py"]
CMD ["--help"]
EOF
```

Build and test:

```bash
# Build
docker build -t ml-train:1.0 .

# Check size
docker images ml-train:1.0

# Test
mkdir -p output
docker run -v $(pwd)/output:/output ml-train:1.0 --output /output/model.joblib

# Verify model was saved
ls -la output/
```

---

## Container Size Comparison

| Approach | Typical Size |
|----------|--------------|
| `python:3.11` + packages | 1.5-2 GB |
| `python:3.11-slim` + packages | 400-600 MB |
| Multi-stage with slim | 150-300 MB |
| Multi-stage + optimization | 100-200 MB |

---

## Summary

In this tutorial, you learned:

- **Multi-stage builds**: Separate build and runtime for smaller images
- **GPU containers**: Use NVIDIA CUDA base images and cuDNN
- **MPI containers**: Enable parallel computing across nodes
- **Optimization**: Layer ordering, caching, and cleanup techniques
- **Security**: Non-root users, no secrets, verified downloads

---

## Next Steps

Continue to **[Containers on HPC Clusters](containers-hpc)** to learn how to:
- Run containers on ACE HPC with Apptainer
- Write SLURM job scripts for containerized workloads
- Manage data with bind mounts
- Optimize performance on the cluster

## Additional Resources

- [Docker Multi-stage Builds](https://docs.docker.com/build/building/multi-stage/)
- [NVIDIA NGC Catalog](https://catalog.ngc.nvidia.com/)
- [mpi4py Documentation](https://mpi4py.readthedocs.io/)
- [Apptainer GPU Support](https://apptainer.org/docs/user/latest/gpu.html)

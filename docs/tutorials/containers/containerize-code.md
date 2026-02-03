---
id: containerize-code
title: Containerize Your Code
sidebar_label: Containerize Your Code
sidebar_position: 2
---

# Containerize Your Code

This tutorial teaches you how to create container images for your applications using Dockerfiles. By the end, you'll be able to package any application into a portable container.

## Learning Objectives

After completing this tutorial, you will be able to:

- Write Dockerfiles from scratch
- Understand Dockerfile instructions and best practices
- Build container images using Docker
- Push images to container registries
- Create reproducible research environments

---

## What is a Dockerfile?

A **Dockerfile** is a text file containing instructions for building a container image. Think of it as a recipe that describes:

1. What base operating system to use
2. What software to install
3. What files to include
4. How to configure the environment
5. What command to run by default

Here's a simple example:

```dockerfile
# Start from a Python base image
FROM python:3.11-slim

# Install dependencies
RUN pip install numpy pandas matplotlib

# Set the working directory
WORKDIR /app

# Copy your code into the container
COPY analysis.py .

# Define the default command
CMD ["python", "analysis.py"]
```

---

## Dockerfile Instructions Reference

| Instruction | Purpose | Example |
|-------------|---------|---------|
| `FROM` | Set the base image | `FROM python:3.11-slim` |
| `RUN` | Execute commands during build | `RUN pip install numpy` |
| `COPY` | Copy files from host to image | `COPY src/ /app/` |
| `ADD` | Copy files (with URL/tar support) | `ADD data.tar.gz /data/` |
| `WORKDIR` | Set working directory | `WORKDIR /app` |
| `ENV` | Set environment variables | `ENV PATH=/app:$PATH` |
| `ARG` | Define build-time variables | `ARG VERSION=1.0` |
| `EXPOSE` | Document exposed ports | `EXPOSE 8080` |
| `CMD` | Default command to run | `CMD ["python", "app.py"]` |
| `ENTRYPOINT` | Configure container executable | `ENTRYPOINT ["python"]` |
| `LABEL` | Add metadata | `LABEL version="1.0"` |
| `USER` | Set the user | `USER appuser` |

---

## Exercise 1: Your First Dockerfile

Let's create a simple Python container step by step.

### Step 1: Create a Project Directory

```bash
mkdir -p ~/my-container
cd ~/my-container
```

### Step 2: Create a Python Script

```bash
cat > hello.py << 'EOF'
#!/usr/bin/env python3
"""Simple hello world script."""

import sys
import platform

def main():
    print("=" * 50)
    print("Hello from inside a container!")
    print("=" * 50)
    print(f"Python version: {sys.version}")
    print(f"Platform: {platform.platform()}")
    print(f"Architecture: {platform.machine()}")
    print("=" * 50)

if __name__ == "__main__":
    main()
EOF
```

### Step 3: Write the Dockerfile

```bash
cat > Dockerfile << 'EOF'
# Use official Python image as base
FROM python:3.11-slim

# Add metadata
LABEL maintainer="your.email@example.com"
LABEL description="My first container"
LABEL version="1.0"

# Set working directory
WORKDIR /app

# Copy the script
COPY hello.py .

# Run the script by default
CMD ["python", "hello.py"]
EOF
```

### Step 4: Build the Image

```bash
# Build the image with a tag
docker build -t my-hello:1.0 .

# List your images
docker images | grep my-hello
```

**Expected output:**
```
my-hello    1.0    abc123def    10 seconds ago    125MB
```

### Step 5: Run the Container

```bash
docker run my-hello:1.0
```

**Expected output:**
```
==================================================
Hello from inside a container!
==================================================
Python version: 3.11.x (main, ...) [GCC ...]
Platform: Linux-...
Architecture: x86_64
==================================================
```

Congratulations! You've built and run your first container.

---

## Understanding Base Images

The `FROM` instruction specifies your starting point. Choosing the right base image is important.

### Common Base Images

| Image | Size | Use Case |
|-------|------|----------|
| `ubuntu:22.04` | ~77MB | General purpose, familiar environment |
| `debian:bookworm-slim` | ~74MB | Stable, minimal Debian |
| `alpine:3.18` | ~7MB | Minimal, security-focused (may have compatibility issues) |
| `python:3.11` | ~1GB | Full Python with build tools |
| `python:3.11-slim` | ~125MB | Python without extras |
| `python:3.11-alpine` | ~50MB | Smallest Python (limited compatibility) |

### Choosing a Base Image

```dockerfile
# For Python data science work
FROM python:3.11-slim

# For R analysis
FROM rocker/r-ver:4.3

# For bioinformatics
FROM ubuntu:22.04

# For machine learning with GPU
FROM nvidia/cuda:12.2.0-runtime-ubuntu22.04
```

**Recommendation**: Start with `-slim` variants for a good balance of size and compatibility.

---

## Exercise 2: Python Data Science Container

Let's build a more realistic container for data science work.

### Step 1: Create Project Files

```bash
mkdir -p ~/datascience-container
cd ~/datascience-container
```

Create a requirements file:

```bash
cat > requirements.txt << 'EOF'
numpy==1.24.3
pandas==2.0.3
matplotlib==3.7.2
seaborn==0.12.2
scikit-learn==1.3.0
EOF
```

Create an analysis script:

```bash
cat > analyze.py << 'EOF'
#!/usr/bin/env python3
"""Sample data analysis script."""

import numpy as np
import pandas as pd
import matplotlib
matplotlib.use('Agg')  # Use non-interactive backend
import matplotlib.pyplot as plt
import seaborn as sns
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier

def main():
    print("Loading iris dataset...")
    iris = load_iris()
    df = pd.DataFrame(iris.data, columns=iris.feature_names)
    df['species'] = iris.target

    print(f"Dataset shape: {df.shape}")
    print(f"\nDataset summary:\n{df.describe()}")

    # Simple ML model
    X_train, X_test, y_train, y_test = train_test_split(
        iris.data, iris.target, test_size=0.2, random_state=42
    )

    clf = RandomForestClassifier(n_estimators=100, random_state=42)
    clf.fit(X_train, y_train)
    accuracy = clf.score(X_test, y_test)

    print(f"\nRandom Forest Accuracy: {accuracy:.2%}")

    # Create a plot
    plt.figure(figsize=(10, 6))
    sns.pairplot(df, hue='species', diag_kind='hist')
    plt.savefig('/output/iris_analysis.png', dpi=150, bbox_inches='tight')
    print("\nPlot saved to /output/iris_analysis.png")

if __name__ == "__main__":
    main()
EOF
```

### Step 2: Write the Dockerfile

```bash
cat > Dockerfile << 'EOF'
# Python data science container
FROM python:3.11-slim

# Metadata
LABEL maintainer="your.email@example.com"
LABEL description="Python data science environment"
LABEL version="1.0"

# Install system dependencies for matplotlib
RUN apt-get update && apt-get install -y --no-install-recommends \
    libgomp1 \
    && rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# Copy and install Python dependencies first (for caching)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY analyze.py .

# Create output directory
RUN mkdir -p /output

# Default command
CMD ["python", "analyze.py"]
EOF
```

### Step 3: Build and Test

```bash
# Build the image
docker build -t datascience:1.0 .

# Create a local output directory
mkdir -p output

# Run with volume mount for output
docker run -v $(pwd)/output:/output datascience:1.0

# Check the output
ls -la output/
```

---

## Working with Files and Dependencies

### Copying Files

```dockerfile
# Copy a single file
COPY script.py /app/

# Copy multiple files
COPY script1.py script2.py /app/

# Copy a directory
COPY src/ /app/src/

# Copy with a different name
COPY local-config.yaml /app/config.yaml

# Copy everything (use .dockerignore to exclude files)
COPY . /app/
```

### Using .dockerignore

Create a `.dockerignore` file to exclude files from the build:

```bash
cat > .dockerignore << 'EOF'
# Git
.git
.gitignore

# Python
__pycache__
*.pyc
*.pyo
.pytest_cache
.venv
venv

# IDE
.vscode
.idea
*.swp

# Data files (usually too large)
*.csv
*.parquet
data/

# Output
output/
*.log

# Docker
Dockerfile
.dockerignore
EOF
```

### Installing System Packages

```dockerfile
# Debian/Ubuntu based images
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    curl \
    git \
    && rm -rf /var/lib/apt/lists/*

# Alpine based images
RUN apk add --no-cache \
    build-base \
    curl \
    git
```

**Best Practice**: Always combine `apt-get update` with `apt-get install` in the same `RUN` command, and clean up afterward.

---

## Exercise 3: R Statistical Container

Let's create a container for R analysis.

### Step 1: Create Project Files

```bash
mkdir -p ~/r-container
cd ~/r-container
```

Create an R script:

```bash
cat > analysis.R << 'EOF'
#!/usr/bin/env Rscript
# Sample R analysis script

library(tidyverse)

cat("=" , rep("=", 48), "=\n", sep="")
cat("R Analysis Container\n")
cat("=" , rep("=", 48), "=\n", sep="")

cat("\nR version:", R.version.string, "\n")
cat("tidyverse version:", as.character(packageVersion("tidyverse")), "\n")

# Create sample data
set.seed(42)
data <- tibble(
  x = rnorm(100),
  y = 2 * x + rnorm(100, sd = 0.5),
  group = sample(c("A", "B"), 100, replace = TRUE)
)

cat("\nDataset summary:\n")
print(summary(data))

# Simple linear model
model <- lm(y ~ x, data = data)
cat("\nLinear model summary:\n")
print(summary(model))

# Create a plot
p <- ggplot(data, aes(x = x, y = y, color = group)) +
  geom_point(alpha = 0.7) +
  geom_smooth(method = "lm", se = TRUE) +
  theme_minimal() +
  labs(title = "Sample Analysis", x = "X Variable", y = "Y Variable")

ggsave("/output/analysis_plot.png", p, width = 10, height = 6, dpi = 150)
cat("\nPlot saved to /output/analysis_plot.png\n")
EOF
```

### Step 2: Write the Dockerfile

```bash
cat > Dockerfile << 'EOF'
# R statistical computing container
FROM rocker/tidyverse:4.3

# Metadata
LABEL maintainer="your.email@example.com"
LABEL description="R statistical analysis environment"
LABEL version="1.0"

# Install additional R packages if needed
RUN R -e "install.packages(c('ggplot2', 'dplyr'), repos='https://cloud.r-project.org')"

# Set working directory
WORKDIR /app

# Copy the analysis script
COPY analysis.R .

# Create output directory
RUN mkdir -p /output

# Default command
CMD ["Rscript", "analysis.R"]
EOF
```

### Step 3: Build and Run

```bash
docker build -t r-analysis:1.0 .
mkdir -p output
docker run -v $(pwd)/output:/output r-analysis:1.0
```

---

## Environment Variables

### Setting Environment Variables

```dockerfile
# Set a single variable
ENV MY_VAR=value

# Set multiple variables
ENV APP_HOME=/app \
    APP_VERSION=1.0 \
    PYTHONUNBUFFERED=1

# Use variables in subsequent commands
WORKDIR $APP_HOME
```

### Using ARG for Build-time Variables

```dockerfile
# Define build argument with default
ARG PYTHON_VERSION=3.11

# Use in FROM
FROM python:${PYTHON_VERSION}-slim

# ARG values don't persist to runtime
ARG BUILD_DATE
LABEL build_date=${BUILD_DATE}

# Convert ARG to ENV if needed at runtime
ARG APP_VERSION=1.0
ENV APP_VERSION=${APP_VERSION}
```

Build with custom arguments:

```bash
docker build --build-arg PYTHON_VERSION=3.10 --build-arg BUILD_DATE=$(date -I) -t myapp:1.0 .
```

---

## Pushing to Container Registries

### Docker Hub

```bash
# Login to Docker Hub
docker login

# Tag your image for Docker Hub
docker tag datascience:1.0 yourusername/datascience:1.0

# Push to Docker Hub
docker push yourusername/datascience:1.0
```

### GitHub Container Registry

```bash
# Login to GitHub Container Registry
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Tag for GHCR
docker tag datascience:1.0 ghcr.io/yourusername/datascience:1.0

# Push
docker push ghcr.io/yourusername/datascience:1.0
```

### Using Your Image on ACE HPC

Once pushed, you can use it on ACE:

```bash
# On ACE HPC
module load apptainer

# Pull from Docker Hub
apptainer pull docker://yourusername/datascience:1.0

# Or from GitHub Container Registry
apptainer pull docker://ghcr.io/yourusername/datascience:1.0
```

---

## Exercise 4: Complete Workflow

Let's put it all together with a complete example.

### Step 1: Create a Bioinformatics Tool Container

```bash
mkdir -p ~/bioinfo-container
cd ~/bioinfo-container
```

Create a simple sequence analysis script:

```bash
cat > seq_analysis.py << 'EOF'
#!/usr/bin/env python3
"""Simple sequence analysis tool."""

import argparse
from collections import Counter

def gc_content(sequence):
    """Calculate GC content of a DNA sequence."""
    sequence = sequence.upper()
    gc = sequence.count('G') + sequence.count('C')
    return gc / len(sequence) * 100 if sequence else 0

def nucleotide_freq(sequence):
    """Calculate nucleotide frequencies."""
    return Counter(sequence.upper())

def main():
    parser = argparse.ArgumentParser(description='Analyze DNA sequences')
    parser.add_argument('--sequence', '-s', type=str, help='DNA sequence to analyze')
    parser.add_argument('--file', '-f', type=str, help='File containing sequence')
    args = parser.parse_args()

    if args.file:
        with open(args.file) as f:
            sequence = ''.join(line.strip() for line in f if not line.startswith('>'))
    elif args.sequence:
        sequence = args.sequence
    else:
        # Default example sequence
        sequence = "ATGCGATCGATCGATCGATCGATCGATCGATCGATCG"

    print("=" * 50)
    print("Sequence Analysis Results")
    print("=" * 50)
    print(f"Sequence length: {len(sequence)} bp")
    print(f"GC content: {gc_content(sequence):.2f}%")
    print("\nNucleotide frequencies:")
    for nuc, count in sorted(nucleotide_freq(sequence).items()):
        print(f"  {nuc}: {count} ({count/len(sequence)*100:.1f}%)")
    print("=" * 50)

if __name__ == "__main__":
    main()
EOF
```

### Step 2: Write the Dockerfile

```bash
cat > Dockerfile << 'EOF'
# Bioinformatics analysis tool
FROM python:3.11-slim

LABEL maintainer="your.email@example.com"
LABEL description="DNA sequence analysis tool"
LABEL version="1.0"

# Create non-root user for security
RUN useradd -m -s /bin/bash biouser

# Set working directory
WORKDIR /app

# Copy application
COPY seq_analysis.py .
RUN chmod +x seq_analysis.py

# Switch to non-root user
USER biouser

# Set entrypoint for flexible usage
ENTRYPOINT ["python", "seq_analysis.py"]

# Default arguments (can be overridden)
CMD ["--help"]
EOF
```

### Step 3: Build

```bash
docker build -t seq-analysis:1.0 .
```

### Step 4: Test Different Usage Patterns

```bash
# Show help
docker run seq-analysis:1.0

# Analyze a sequence directly
docker run seq-analysis:1.0 --sequence "ATGCGATCGATCGATCG"

# Analyze from a file
echo ">sample" > test.fasta
echo "ATGCGATCGATCGATCGATCGATCGATCG" >> test.fasta
docker run -v $(pwd):/data seq-analysis:1.0 --file /data/test.fasta
```

### Step 5: Push and Use on ACE

```bash
# Tag and push
docker tag seq-analysis:1.0 yourusername/seq-analysis:1.0
docker push yourusername/seq-analysis:1.0

# On ACE HPC:
# module load apptainer
# apptainer exec docker://yourusername/seq-analysis:1.0 --sequence "ATGCGATCG"
```

---

## Dockerfile Best Practices Summary

### 1. Order Instructions by Change Frequency

Put instructions that change rarely at the top:

```dockerfile
FROM python:3.11-slim

# System deps (rarely change)
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Python deps (change occasionally)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Application code (changes frequently)
COPY src/ /app/src/
```

### 2. Combine RUN Commands

```dockerfile
# Bad: Multiple layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# Good: Single layer
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

### 3. Use Specific Tags

```dockerfile
# Bad: Tag can change
FROM python:latest

# Good: Specific version
FROM python:3.11.4-slim
```

### 4. Don't Include Secrets

```dockerfile
# NEVER do this
ENV API_KEY=secret123
COPY credentials.json /app/

# Instead, mount secrets at runtime
# docker run -v ~/.aws:/root/.aws:ro myimage
```

### 5. Add Metadata

```dockerfile
LABEL maintainer="email@example.com"
LABEL version="1.0"
LABEL description="What this container does"
```

---

## Troubleshooting Common Issues

### Build Fails: "Package not found"

```dockerfile
# Always update package lists first
RUN apt-get update && apt-get install -y package-name
```

### Build Fails: "Permission denied"

```dockerfile
# Make scripts executable
COPY script.sh .
RUN chmod +x script.sh

# Or use proper ownership
COPY --chown=user:user files/ /app/
```

### Image Too Large

```dockerfile
# Use slim base images
FROM python:3.11-slim  # Not python:3.11

# Clean up in the same layer
RUN apt-get update \
    && apt-get install -y --no-install-recommends pkg \
    && rm -rf /var/lib/apt/lists/*

# Use --no-cache-dir for pip
RUN pip install --no-cache-dir -r requirements.txt
```

### Container Exits Immediately

```dockerfile
# Ensure CMD/ENTRYPOINT keeps running
CMD ["python", "-u", "app.py"]  # -u for unbuffered output

# Or for debugging, keep container running
CMD ["tail", "-f", "/dev/null"]
```

---

## Summary

In this tutorial, you learned:

- How to write Dockerfiles using common instructions
- How to choose appropriate base images
- How to manage dependencies and files
- How to build and tag images
- How to push images to registries
- Best practices for efficient, secure containers

---

## Next Steps

Continue to **[Advanced Build Topics](advanced-builds)** to learn about:
- Multi-stage builds for smaller images
- GPU-enabled containers
- MPI for parallel computing
- Optimization techniques

## Additional Resources

- [Dockerfile Reference](https://docs.docker.com/engine/reference/builder/)
- [Docker Build Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Docker Hub](https://hub.docker.com/) - Find and share container images

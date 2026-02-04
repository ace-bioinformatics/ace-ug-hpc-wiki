---
id: intro
title: Welcome to Containers @ ACE!
sidebar_label: What are Containers ?
sidebar_position: 1
---

# Welcome to Containers @ ACE!

Welcome to the ACE HPC Container Tutorial Series. This guide will teach you everything you need to know about using containers for your research computing workflows.

## What are Containers?

Containers are lightweight, portable packages that bundle an application together with all its dependencies—libraries, tools, and runtime environments—into a single unit that runs consistently across different computing environments.

Think of a container as a **shipping container for software**: just as shipping containers standardized global trade by providing a uniform way to transport goods regardless of what's inside, software containers standardize how applications are packaged and deployed regardless of the underlying infrastructure.

```
┌───────────────────────────────────────┐
│           Your Container              │
│  ┌─────────────────────────────────┐  │
│  │     Your Application            │  │
│  │  ┌─────────┐ ┌─────────┐        │  │
│  │  │ Python  │ │ NumPy   │        │  │
│  │  │ 3.11    │ │ 1.24    │ ...    │  │
│  │  └─────────┘ └─────────┘        │  │
│  └─────────────────────────────────┘  │
│         Operating System Layer        │
│           (Ubuntu 22.04)              │
└───────────────────────────────────────┘
         Runs anywhere: laptop,
         cloud, HPC cluster
```

## Why Use Containers?

### The Problem: "It Works on My Machine"

Have you ever experienced any of these situations?

- Your code works perfectly on your laptop but fails on the cluster
- A colleague can't reproduce your results because they have different library versions
- You need software that requires an older version of a library than what's installed
- Setting up your analysis environment takes hours of installing dependencies

### The Solution: Containers

Containers solve these problems by providing:

| Benefit | Description |
|---------|-------------|
| **Reproducibility** | Run the exact same software environment anywhere, every time |
| **Portability** | Move your entire workflow between laptop → cloud → HPC seamlessly |
| **Isolation** | Keep your application's dependencies separate from other software |
| **Collaboration** | Share your complete environment with colleagues, not just code |
| **Version Control** | Maintain multiple versions of software without conflicts |

## Containers vs. Virtual Machines

You might be familiar with Virtual Machines (VMs). Containers are similar but more lightweight:

```
     Virtual Machines                    Containers
┌─────────────────────────┐      ┌─────────────────────────┐
│   App A  │   App B      │      │   App A  │   App B      │
├──────────┼──────────────┤      ├──────────┼──────────────┤
│ Guest OS │  Guest OS    │      │  Libs A  │   Libs B     │
├──────────┴──────────────┤      ├──────────┴──────────────┤
│       Hypervisor        │      │    Container Runtime    │
├─────────────────────────┤      ├─────────────────────────┤
│        Host OS          │      │        Host OS          │
├─────────────────────────┤      ├─────────────────────────┤
│       Hardware          │      │       Hardware          │
└─────────────────────────┘      └─────────────────────────┘
     Heavy (GBs)                      Light (MBs)
     Minutes to start                 Seconds to start
```

| Aspect | Virtual Machines | Containers |
|--------|------------------|------------|
| Size | Gigabytes | Megabytes |
| Startup | Minutes | Seconds |
| Performance | ~5-10% overhead | Near-native |
| Isolation | Complete | Process-level |
| Resource usage | High | Low |

## Container Technologies

### Docker: The Industry Standard

**Docker** is the most widely used container platform. It provides:
- Simple syntax for building containers (Dockerfile)
- Massive ecosystem of pre-built images (Docker Hub)
- Great tooling for development

However, Docker requires root (administrator) privileges, which makes it unsuitable for shared HPC systems where users don't have root access.

### Apptainer: Containers for HPC

**Apptainer** (formerly Singularity) is a container platform designed specifically for HPC environments:

- **No root required**: Runs as your normal user
- **HPC-integrated**: Works seamlessly with SLURM job schedulers
- **Docker-compatible**: Can run Docker images directly
- **Performance**: Minimal overhead, direct hardware access
- **Security**: Designed for multi-user systems

**On ACE HPC, we use Apptainer to run containers.**

The good news: You can build containers using familiar **Dockerfiles**, then run them on ACE using Apptainer!

## The Container Workflow on ACE HPC

Here's the typical workflow for using containers on the ACE HPC:

```
┌───────────────────────────────────────────────────────────┐
│                    YOUR WORKSTATION                       │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐  │
│  │   Write     │ --> │   Build     │ --> │   Push to   │  │
│  │ Dockerfile  │     │   Image     │     │  Registry   │  │
│  └─────────────┘     └─────────────┘     └─────────────┘  │
└───────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────┐
│                       ACE HPC                             │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐  │
│  │ Pull Image  │ --> │ Convert to  │ --> │ Run with    │  │
│  │ (apptainer) │     │    SIF      │     │   SLURM     │  │
│  └─────────────┘     └─────────────┘     └─────────────┘  │
└───────────────────────────────────────────────────────────┘
```

- **Step 1**: Write a Dockerfile describing your environment
- **Step 2**: Build the image (on your machine or a build service)
- **Step 3**: Push to a registry (Docker Hub, GitHub Container Registry)
- **Step 4**: Pull the image on ACE HPC using Apptainer
- **Step 5**: Run your containerized workflow with SLURM

## Tutorial Roadmap

This tutorial series is organized into four main sections:

### Welcome to Containers @ ACE!
Introduction to container concepts and the ACE HPC container ecosystem.

### [Containerize Your Code](containerize-code)
Learn to create your own containers:
- Writing Dockerfiles
- Building images
- Working with registries
- Your first containerized application

### [Containers on HPC Clusters](containers-hpc)
Run containers on ACE HPC:
- Using Apptainer
- SLURM job scripts for containers
- Data management and bind mounts
- Performance considerations

### [Advanced Build Topics](advanced-builds)
Master advanced container techniques:
- Multi-stage builds for smaller images
- GPU-enabled containers
- MPI for parallel computing
- Optimization and best practices

## Prerequisites

Before starting these tutorials, you should have:

- [ ] Basic familiarity with the Linux command line
- [ ] Basic understanding of your programming language (Python, R, etc.)

For building containers (Section 2), you'll also benefit from:
- [ ] Docker installed on your local machine ([Install Docker](https://docs.docker.com/get-docker/))
- [ ] A Docker Hub account ([Sign up free](https://hub.docker.com/signup))

## Quick Start: Run Your First Container

Let's run a container right now to see how it works! Log into ACE HPC and try:

```bash
# Load the Apptainer module
module load apptainer

# Run a simple Python container
apptainer exec docker://python:3.11-slim python --version
```

**Expected output:**
```
Python 3.11.x
```

Congratulations! You just ran Python inside a container. The container was automatically downloaded from Docker Hub and executed using Apptainer.

Let's try something more interesting:

```bash
# Run Python code inside the container
apptainer exec docker://python:3.11-slim python -c "
import sys
print('Hello from inside a container!')
print(f'Python version: {sys.version}')
print(f'Running on: {sys.platform}')
"
```

## Common Container Use Cases

### Bioinformatics

```bash
# Run BLAST from a container
apptainer exec docker://quay.io/biocontainers/blast:2.14.0--h7d5a4b4_1 \
    blastn -version
```

### Machine Learning

```bash
# Use a complete ML environment
apptainer exec docker://tensorflow/tensorflow:latest \
    python -c "import tensorflow as tf; print(tf.__version__)"
```

### R Statistical Analysis

```bash
# Run R with tidyverse
apptainer exec docker://rocker/tidyverse:4.3 \
    R --version
```

### Custom Research Software

```bash
# Your own containerized application
apptainer exec my-research-tool.sif ./analyze_data.py
```

## References

- **Apptainer Documentation**: [apptainer.org/docs](https://apptainer.org/docs/user/latest/)
- **Docker Documentation**: [docs.docker.com](https://docs.docker.com/)
- **BioContainers**: [biocontainers.pro](https://biocontainers.pro/) for bioinformatics tools
- **Quay.io** [quay.io](https://quay.io/) Container Registry

---

## Next Steps

Ready to create your own containers? Continue to **[Containerize Your Code](containerize-code)** to learn how to write Dockerfiles and build container images.

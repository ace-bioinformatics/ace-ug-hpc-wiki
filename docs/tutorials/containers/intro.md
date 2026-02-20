---
id: intro
title: Introduction to Containers
sidebar_label: What are Containers?
sidebar_position: 1
---

# Introduction to Containers

## What is a Container?

A container is a lightweight, portable package that bundles an application together with everything it needs to run — libraries, tools, system packages, and configuration — into a single unit that behaves the same way on any machine.

Without containers, getting software to run on a new machine means manually installing the right versions of every dependency and hoping nothing conflicts with what's already installed. With containers, you package the entire environment once and run it anywhere.

```
┌─────────────────────────────────────┐
│         Your Container              │
│  ┌───────────────────────────────┐  │
│  │     Your Application          │  │
│  │  ┌─────────┐ ┌─────────┐      │  │
│  │  │ Python  │ │ NumPy   │      │  │
│  │  │ 3.11    │ │ 1.24    │ ...  │  │
│  │  └─────────┘ └─────────┘      │  │
│  └───────────────────────────────┘  │
│       Operating System Layer        │
│         (Ubuntu 22.04)              │
└─────────────────────────────────────┘
       Runs the same everywhere:
       laptop, cloud, HPC cluster
```

A container image is a read-only template. When you "run" an image, you create a container — a live instance of that template. You can run many containers from the same image, and each one is isolated from the others.

:::info Why should you care?
If you've ever said "it works on my machine" and then spent hours debugging it on the cluster, containers eliminate that problem entirely. The environment that works on your laptop is the same environment that runs on ACE HPC.
:::

## Containers vs. Virtual Machines

You may be familiar with Virtual Machines (VMs). Both VMs and containers provide isolation, but they work differently:

- A **Virtual Machine** includes an entire guest operating system on top of a hypervisor. This makes VMs heavy (gigabytes), slow to start (minutes), and resource-intensive.
- A **Container** shares the host operating system's kernel and only packages the application and its dependencies. This makes containers lightweight (megabytes), fast to start (seconds), and nearly as performant as running natively.

For HPC workloads where performance matters, containers add negligible overhead compared to running software directly on the host.

## Docker and Apptainer

Two container platforms matter for HPC work:

### Docker

[Docker](https://www.docker.com/) is the industry standard for building and distributing container images. It provides:

- A simple build syntax (the **Dockerfile**)
- A massive ecosystem of pre-built images on [Docker Hub](https://hub.docker.com/)
- Excellent developer tooling

However, Docker requires root (administrator) privileges to run. On shared HPC systems where users don't have root access, Docker cannot be used directly.

### Apptainer

[Apptainer](https://apptainer.org/) (formerly Singularity) was built specifically for HPC. It solves the root privilege problem:

- **Runs as your normal user** — no root required
- **Integrates with SLURM** — works seamlessly in batch job scripts
- **Runs Docker images directly** — pull any Docker image and run it with Apptainer
- **Near-native performance** — minimal overhead, direct access to host hardware (GPUs, high-speed networks)

**On ACE HPC, we use Apptainer to run containers.**

The practical workflow is: build your containers with Docker on your workstation, push them to a registry, then pull and run them with Apptainer on ACE HPC.

## The Container Workflow on ACE HPC

```
  Your Workstation (Docker)              ACE HPC (Apptainer + SLURM)
 ┌──────────────────────────┐          ┌──────────────────────────┐
 │ 1. Write Dockerfile      │          │ 4. Pull image            │
 │ 2. Build image           │  ──────> │ 5. Convert to .sif       │
 │ 3. Push to registry      │          │ 6. Run with SLURM        │
 └──────────────────────────┘          └──────────────────────────┘
                    │
              Docker Hub or
              GitHub Container
              Registry
```

1. **Write** a Dockerfile describing your software environment
2. **Build** the container image on your workstation (where you have Docker)
3. **Push** the image to a container registry (Docker Hub, GitHub Container Registry)
4. **Pull** the image on ACE HPC using Apptainer
5. **Convert** — Apptainer automatically converts Docker images to its native `.sif` format
6. **Run** your containerized application through SLURM job scripts

## Quick Start: Run Your First Container

Log into ACE HPC and try this:

```bash
# Load the Apptainer module
module load apptainer

# Pull a Python container from Docker Hub
apptainer pull docker://python:3.11-slim

# Run Python inside the container
apptainer exec python_3.11-slim.sif python -c "
import sys
print('Hello from inside a container!')
print(f'Python version: {sys.version}')
import os
print(f'User: {os.environ.get(\"USER\", \"unknown\")}')
print(f'Hostname: {os.uname().nodename}')
"
```

Notice two things: (1) you didn't install Python — it came from the container, and (2) your username and host environment are visible inside the container. This is Apptainer's design — it runs as *you*, not as root.

## Prerequisites

To follow these tutorials, you'll need:

- **Basic Linux command-line skills** — navigating directories, editing files, running commands
- **A programming language** — the examples use Python, but the concepts apply to any language

For building containers (covered in the next section), you'll also need:

- [Docker Desktop](https://docs.docker.com/get-docker/) installed on your local machine
- A free [Docker Hub](https://hub.docker.com/signup) account

## Tutorial Roadmap

This tutorial series has three sections:

| Section | What You'll Learn |
|---------|-------------------|
| [Containerize Your Code](containerize-code) | Write a Dockerfile, build an image, push it to a registry |
| [Advanced Build Topics](advanced-builds) | Multi-stage builds, multi-architecture builds |
| [Containers on HPC Clusters](containers-hpc) | Apptainer on ACE HPC, SLURM job scripts, MPI and GPU containers |

## References

- [Apptainer User Guide](https://apptainer.org/docs/user/latest/)
- [Docker Documentation](https://docs.docker.com/)
- [Docker Hub](https://hub.docker.com/) — public registry of container images
- [BioContainers](https://biocontainers.pro/) — pre-built containers for bioinformatics tools
- [NVIDIA NGC Catalog](https://catalog.ngc.nvidia.com/) — GPU-optimized containers

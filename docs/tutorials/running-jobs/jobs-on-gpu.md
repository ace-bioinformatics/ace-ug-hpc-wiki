---
title: Using the GPU Queue
sidebar_label: Using GPUs
description: How to request access to the GPU node and run interactive and batch jobs on it.
---

# Using the GPU Queue

The ACE HPC Cluster provides access to a single **NVIDIA A100 80GB PCIe** GPU node (`kla-ac-gpu-01`), partitioned using NVIDIA Multi-Instance GPU (MIG) technology. This guide explains how to request access and how to run both interactive and batch jobs on the GPU.

---

## 1. Requesting GPU Access

GPU access is not granted by default. It is provisioned only for users who have a demonstrated workload requirement (e.g. deep learning training, GPU-accelerated bioinformatics tools).

### Steps

1. Send an email to **support@ace-bioinformatics.org** requesting GPU queue access.
2. Include the following details in your request:
   - Your HPC username
   - A brief description of the workload requiring GPU acceleration
   - The software/framework you intend to use (e.g. PyTorch, TensorFlow, AlphaFold2)
3. The ACE HPC team will review your request and, once approved, will add you to the appropriate Active Directory group and Slurm GPU account.
4. You will receive a confirmation email once access has been provisioned.

:::info
Provisioning is typically completed within 1–2 business days of an approved request.
:::

---

## 2. GPU Hardware Overview

| Spec | Value |
|---|---|
| GPU Model | NVIDIA A100 80GB PCIe |
| Driver Version | 570.211.01 |
| CUDA Version | 12.8 |
| Total Memory | 80GB (81920 MiB) |
| MIG Mode | Enabled |

The A100 is split into **two MIG instances**, each with its own isolated compute and memory allocation:

| MIG Instance | GRES Name | Memory |
|---|---|---|
| Instance 1 | `a100_3g40gb` | 40GB |
| Instance 2 | `a100_3g40gb` | 40GB |

:::warning Only One MIG Instance Per User
The GPU node has only **two MIG instances available in total**. Because resources are limited, **a single user may only request and use one MIG instance at a time.**

You **cannot** request both `a100_3g40gb` instances simultaneously for the same job or across concurrent jobs. If you need more GPU throughput, queue a second job — it will run once your first job completes or once the other instance is free.
:::

---

## 3. Interactive GPU Jobs

Interactive jobs are useful for testing scripts, debugging code, or exploratory work before submitting a full batch job.

### Requesting an Interactive Session

```bash
srun -p gpu --gres=gpu:a100_3g40gb:1 --pty bash
```

This will:
- Submit a request to the `gpu` partition
- Allocate **one** MIG instance (`:1`)
- Drop you into an interactive bash shell on `kla-ac-gpu-01` once resources are available

### Verifying Your GPU Allocation

Once inside the session, confirm your GPU allocation:

```bash
nvidia-smi
```

You should see your allocated MIG instance listed under **MIG devices**, along with any processes you start.

### Example Workflow

```bash
# Request an interactive session
srun -p gpu --gres=gpu:a100_3g40gb:1 --time=02:00:00 --pty bash

# Activate your environment
conda activate myenv

# Run your script interactively
python train.py --epochs 5
```

:::tip
Always set a `--time` limit on interactive sessions (e.g. `--time=02:00:00`) so the MIG instance is automatically released back to the queue when you're done, freeing it up for other users.
:::

### Exiting

```bash
exit
```

This releases the MIG instance back to the partition immediately.

---

## 4. Batch GPU Jobs

For longer-running or unattended workloads, submit a batch job using `sbatch`.

### Example Batch Script

Create a script, e.g. `gpu_job.sh`:

```bash
#!/bin/bash
#SBATCH --job-name=gpu_training
#SBATCH --partition=gpu
#SBATCH --gres=gpu:a100_3g40gb:1
#SBATCH --output=gpu_job_%j.log
#SBATCH --time=08:00:00
#SBATCH --mem=32G
#SBATCH --cpus-per-task=4

# Activate your environment
source ~/.bashrc
conda activate myenv

# Run your workload
python train.py --epochs 50 --batch-size 64
```

Submit the job:

```bash
sbatch gpu_job.sh
```

---

## 5. Best Practices

:::tip Recommended Practices
- Always specify `--gres=gpu:a100_3g40gb:1` — never request more than `:1` instance per job.
- Set realistic `--time` limits so unused allocations are released promptly.
- Test your code in an interactive session before submitting a long-running batch job.
- Monitor GPU utilisation with `nvidia-smi` during your session to confirm your workload is actually using the GPU.
- Release interactive sessions (`exit`) as soon as you're done — don't leave idle sessions holding a MIG instance.
:::

:::danger Common Mistake
Requesting `--gres=gpu:a100_3g40gb:2` will fail or hold your job indefinitely in the queue, since this would require **both** MIG instances at once — which conflicts with the single-instance-per-user policy. Always request `:1`.
:::

---

## 6. Troubleshooting

| Issue | Likely Cause | Resolution |
|---|---|---|
| Job stuck in `PD` (pending) state | Both MIG instances currently in use by other users | Wait for an instance to free up, or check `squeue -p gpu` to see current usage |
| `srun: Requested partition configuration not available now` | No GPU account/QOS provisioned for your user | Confirm your access was provisioned — contact support@ace-bioinformatics.org |
| `nvidia-smi` shows no MIG device | GPU allocation not yet active or job not yet started on the node | Confirm with `squeue -u $USER` that your job has a `NODELIST` assigned |

---

## 7. Getting Help

If you experience issues accessing or using the GPU queue, contact:

**support@ace-bioinformatics.org**

Include your username, job ID (if applicable), and a description of the issue.
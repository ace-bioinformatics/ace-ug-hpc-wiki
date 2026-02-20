---
id: knl-nodes
title: Using KNL Nodes on ACE
sidebar_label: KNL Node Features
---

The ACE HPC Cluster includes **Intel Xeon Phi 7250** nodes — commonly called **KNL nodes** (Knights Landing is the processor's codename). These nodes have a fundamentally different architecture from standard compute nodes and can deliver significant performance gains for the right kinds of workloads — but only if your job script requests the right configuration.

This guide explains what KNL is, how its special features work, and how to request them from a Slurm job script.

---

## What Makes KNL Different?

A standard server CPU trades core count for clock speed: you get fewer cores, but each one is fast. KNL flips that trade-off entirely.

| Feature | Standard Node | KNL Node |
|---|---|---|
| Cores | 16–28 fast cores | 68 slower cores |
| Hardware threads | ~56 total | **272 total** (4 per core) |
| Memory | DDR4 only | DDR4 + MCDRAM (see below) |
| Vector width | AVX-256 (256-bit) | **AVX-512 (512-bit)** |

The design philosophy is that for scientific computing — genomic alignment, Monte Carlo simulations, matrix operations — many parallel workers processing data in wide chunks often outperforms a handful of fast cores working sequentially.

Each KNL core also supports **4 hardware threads**, meaning a single node can have 272 concurrent threads in flight. This is ideal for applications that can be parallelised across many tasks simultaneously.

:::info
KNL nodes are not faster at every task. If your program is largely single-threaded or has poor memory access patterns, a standard node will serve you better. KNL shines when the workload is **massively parallel** and **bandwidth-hungry**.
:::

---

## MCDRAM: The Fast On-Package Memory

This is the most distinctive feature of KNL nodes. Every KNL node has two memory tiers:

- **DDR4 RAM** — 96 GB of standard main memory. Large, but relatively slow to stream data through.
- **MCDRAM (Multi-Channel DRAM)** — 16 GB of ultra-fast memory physically mounted on the processor package itself, delivering **4–5× the memory bandwidth** of DDR4.

Think of DDR4 as a large warehouse across town and MCDRAM as a small, blazing-fast stockroom right at your workstation. The stockroom holds less, but you can move things in and out almost instantly.

Because MCDRAM is fast but small (only 16 GB), how it is used matters enormously. There are three configurable modes:

### Cache Mode

MCDRAM acts as a very large, hardware-managed cache sitting between the CPU and DDR4. Your application does not need any code changes — data that is accessed repeatedly gets pulled into MCDRAM automatically.

**This is the safest default** and works well for most jobs.

### Flat Mode

MCDRAM appears as a separate, addressable memory region (a separate NUMA node). Your program — or a tool like `numactl` — can explicitly place specific arrays or data structures into MCDRAM for maximum bandwidth.

This gives the highest possible performance, but only if your most-accessed data fits within 16 GB. It requires more careful programming or job setup.

### Hybrid Mode

A portion of MCDRAM acts as cache and the remainder is addressable memory. This is a middle-ground option when only some of your data fits in 16 GB but you still want cache benefits for the rest.

---

## Cluster Modes: How Cores Are Grouped

KNL's 68 cores are physically arranged in pairs called **tiles**, which are then grouped into regions. **Cluster mode** controls how those regions manage memory access and cache coherency — the mechanism that ensures all cores see consistent data when sharing memory.

### Quadrant Mode (Recommended)

The chip is logically divided into four virtual quadrants. Each core preferentially accesses memory within its own quadrant, reducing traffic across the chip. To your software the node still appears as a single memory node, so no NUMA-aware programming is required.

This is the **default at ACE** and the right choice for most jobs.

### All-to-All Mode

All cores access all memory equally with no locality preference. Simple to reason about, but cache coherency traffic spreads across the whole chip, which can reduce performance for bandwidth-intensive work.

### SNC-2 Mode (Sub-NUMA Clustering)

The chip is split into 2 true NUMA sub-nodes. Software that is explicitly written to be NUMA-aware can benefit by pinning tasks to specific sub-nodes and keeping their data nearby.

### SNC-4 Mode (Disabled at ACE)

SNC-4 divides the chip into 4 NUMA sub-nodes — theoretically the best locality. However, **this mode is not available on ACE** due to system instability when booting nodes in this configuration. Requesting it will result in your job not being scheduled.

---

## Bioinformatics Workloads That Benefit from KNL

KNL nodes are particularly well-matched to bioinformatics pipelines that involve:

**Genome-wide alignment at scale** — Tools like BWA-MEM2, Bowtie2, and HISAT2 are I/O and compute intensive. When aligning large cohorts simultaneously, the high thread count and MCDRAM bandwidth reduce wall time significantly compared to a standard node.

**Variant calling across large cohorts** — GATK's HaplotypeCaller and similar tools process large genomic regions by walking arrays of read data. MCDRAM in cache mode can keep the hottest reference data close to the cores, reducing latency.

**Sequence assembly** — Assemblers like SPAdes build and traverse large graphs in memory. The wide memory bandwidth of MCDRAM is directly useful here, especially when intermediate data fits within 16 GB and flat mode can be used.

**Monte Carlo and population genetics simulations** — Tools like PLINK, STRUCTURE, or custom simulation scripts that run thousands of independent iterations map naturally onto KNL's thread count. Each iteration can be assigned its own hardware thread.

**Protein structure prediction pre-processing** — Data preparation pipelines that operate on large arrays of atomic coordinates or contact maps benefit from AVX-512 vector operations when the underlying library is compiled to use them.

:::tip
Before running a full dataset on a KNL node, test with a smaller input on both a standard node and a KNL node. Time both runs. KNL will only be beneficial if your tool can actually exploit the available parallelism.
:::

---

## Requesting KNL Features in Your Slurm Script

KNL configuration is requested through the `--constraint` directive in your Slurm script. The constraint combines a **cluster mode** and a **memory mode** using the `&` operator.

### Constraint Syntax

```
#SBATCH --constraint="<cluster_mode>&<memory_mode>"
```

### Available Combinations

| Constraint | Cluster Mode | MCDRAM Mode | When to use |
|---|---|---|---|
| `quadrant&cache` | Quadrant | Cache | Most jobs — good default |
| `quadrant&flat` | Quadrant | Flat | Data fits in 16 GB, want max bandwidth |
| `quadrant&hybrid` | Quadrant | Hybrid | Mixed workloads |
| `alltoall&cache` | All-to-All | Cache | Simple fallback |
| `snc2&cache` | SNC-2 | Cache | NUMA-aware software only |

:::info
**Node reboot time:** KNL nodes can be reconfigured between jobs, but changing the cluster or memory mode requires rebooting the node — a process that takes approximately **15 minutes**. This reboot time is counted against your job's allocated wall time. Request a common configuration (such as `quadrant&cache`) when possible to avoid this delay.
:::

---

## Complete Example: RNA-Seq Alignment with HISAT2 on a KNL Node

The following job script aligns paired-end RNA-seq reads against a reference genome using HISAT2, taking advantage of KNL's high thread count and MCDRAM cache for reference index access.

```bash
#!/bin/bash

#SBATCH --job-name=hisat2_knl            # Job name
#SBATCH --output=hisat2_knl_%j.out       # Standard output (%j = job ID)
#SBATCH --error=hisat2_knl_%j.err        # Standard error
#SBATCH --time=04:00:00                  # Wall time: 4 hours
#SBATCH --nodes=1                        # Single KNL node
#SBATCH --ntasks=1                       # One task (multi-threaded, not MPI)
#SBATCH --cpus-per-task=64              # Use 64 of the 68 available cores
#SBATCH --mem=64G                        # Request 64 GB of DDR4 RAM
#SBATCH --constraint="quadrant&cache"    # KNL: quadrant cluster mode, MCDRAM as cache

# --- Environment setup ---
module load hisat2/2.2.1
module load samtools/1.17

# --- Variables ---
INDEX=/scratch/$USER/reference/grch38_index   # Path to HISAT2 genome index
READS_1=/scratch/$USER/data/sample_R1.fastq.gz
READS_2=/scratch/$USER/data/sample_R2.fastq.gz
OUTDIR=/scratch/$USER/results/hisat2_knl

mkdir -p "$OUTDIR"

# --- Alignment ---
# -p 64: use all 64 requested CPUs
# MCDRAM cache mode will automatically cache the genome index accesses
hisat2 \
  -p 64 \
  -x "$INDEX" \
  -1 "$READS_1" \
  -2 "$READS_2" \
  --dta \
  -S "$OUTDIR/aligned.sam"

# --- Convert, sort, and index the output ---
samtools view -@ 64 -bS "$OUTDIR/aligned.sam" \
  | samtools sort -@ 64 -o "$OUTDIR/aligned_sorted.bam"

samtools index "$OUTDIR/aligned_sorted.bam"

echo "Alignment complete. Output: $OUTDIR/aligned_sorted.bam"
```

### What this script does

- **`--constraint="quadrant&cache"`** — Requests a KNL node in quadrant cluster mode with MCDRAM operating as a transparent cache. The genome index, which HISAT2 reads repeatedly during alignment, will be cached in MCDRAM automatically, reducing memory bottleneck.
- **`--cpus-per-task=64`** — Requests 64 cores. Four are left free for the operating system, which is a common practice on KNL to avoid resource contention.
- **`--mem=64G`** — Requests 64 GB of DDR4. The genome index and read data live here; hot regions are pulled into MCDRAM cache automatically.
- The `samtools` steps are also parallelised with `-@ 64` to take advantage of the available cores during post-processing.

---

## Flat Mode Example: Explicit MCDRAM Placement

If your working dataset fits within 16 GB, you can use flat mode and explicitly place it in MCDRAM using `numactl`. This requires knowing the NUMA node ID of the MCDRAM region, which on KNL flat-mode nodes is typically node 1.

```bash
#!/bin/bash

#SBATCH --job-name=plink_knl_flat
#SBATCH --output=plink_knl_%j.out
#SBATCH --error=plink_knl_%j.err
#SBATCH --time=02:00:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=64
#SBATCH --mem=32G
#SBATCH --constraint="quadrant&flat"     # Flat mode: MCDRAM is addressable memory

module load plink/1.90

# numactl --membind=1 places memory allocations onto NUMA node 1 (MCDRAM)
# This is only effective if your dataset fits within 16 GB
numactl --membind=1 plink \
  --bfile /scratch/$USER/data/cohort \
  --assoc \
  --out /scratch/$USER/results/plink_knl \
  --threads 64
```

:::tip
Verify that your data fits in MCDRAM before using flat mode. If your program tries to allocate more than 16 GB on NUMA node 1, it will spill over to DDR4 anyway — and you will have gained nothing over cache mode while adding complexity.
:::

---

## Checking Available KNL Nodes

To see which KNL nodes are currently available and what configurations they are in, use `sinfo`:

```bash
sinfo -o "%n %f %T" | grep knl
```

This prints node name, active features (including current cluster and memory mode), and state. A node already configured in `quadrant&cache` can start your job immediately; a node in a different configuration will need to reboot first.

---

## Summary

| Concept | Plain explanation |
|---|---|
| KNL | Intel Xeon Phi 7250 — 68 cores, 272 threads, designed for parallel workloads |
| MCDRAM | 16 GB of ultra-fast on-package memory, 4–5× the bandwidth of DDR4 |
| Cache mode | MCDRAM used automatically as a large cache — no code changes needed |
| Flat mode | MCDRAM exposed as addressable memory — maximum control, requires care |
| Hybrid mode | Part cache, part addressable — a middle-ground option |
| Quadrant mode | Recommended cluster mode — good locality, software sees one NUMA node |
| All-to-All mode | Simpler but potentially slower due to cross-chip traffic |
| SNC-2 mode | Two NUMA sub-nodes — only useful for NUMA-aware software |
| SNC-4 mode | Disabled at ACE due to boot instability |
| Node reboot | Changing configuration takes ~15 minutes, counted in your wall time |

---

## References

- [Intel Xeon Phi 7250 Product Page](https://www.intel.com/content/www/us/en/products/sku/94033/intel-xeon-phi-processor-7250-16gb-1-40-ghz-68-core/specifications.html)
- [SLURM `--constraint` documentation](https://slurm.schedmd.com/sbatch.html#OPT_constraint)
- [numactl manual](https://linux.die.net/man/8/numactl)
- [HISAT2 documentation](https://daehwankimlab.github.io/hisat2/manual/)

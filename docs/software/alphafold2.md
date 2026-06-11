---
id: alphafold2
title: AlphaFold2 on the ACE HPC
sidebar_label: AlphaFold2
---

# Running `AlphaFold2` On the ACE HPC Cluster

We have written a streamlined, user-friendly wrapper script (`run_af2.sh`) to run AlphaFold v2 via Singularity on the ACE-Uganda HPC cluster using the SLURM workload manager. 

This script simplifies the process of submitting AlphaFold jobs by providing a clean command-line interface and automatically routing the correct background databases depending on whether you are predicting a monomer or a multimer complex.

## ✨ Features

- **SLURM Integration:** Pre-configured for optimal resource allocation on the GPU partition.
- **Singularity Support:** Runs AlphaFold2 seamlessly inside a Singularity container (`alphafold_gpu.sif`).
- **Dynamic Database Routing:** Automatically switches between background genetic tracking databases based on the model preset.
- **Simple CLI:** Easy-to-use flags for input, output, and model parameters.

## 🚀 Usage

Submit the script using `sbatch` with the required parameters. See the [parameters](#parameters) section below for details.

### Monomer Prediction (Default)

To run a single protein sequence (monomer):

```bash
sbatch /etc/ace-data/alphafold/run_af2.sh --input /path/to/target.fasta --output /path/to/results
```

> [!NOTE]
> If no preset is specified, the script automatically defaults to `monomer` mode.

### Multimer Prediction

To run a protein complex (multimer):

```bash
sbatch /etc/ace-data/alphafold/run_af2.sh --input /path/to/complex.fasta --output /path/to/results --preset multimer
```

## ⚙️ Parameters

| Argument | Short Flag | Description | Required? |
| :--- | :--- | :--- | :--- |
| `--input` | `-i` | **Absolute path** to the input FASTA file containing your protein sequence(s). | Yes |
| `--output` | `-o` | **Absolute path** to the directory where AlphaFold results will be saved. | Yes |
| `--preset` | `-m` | Specify the model type: `monomer` or `multimer`. Default is `monomer`. | No |

### 📄 Input (`--input`)
The input must be a standard FASTA file:
- For **monomers**, the file should contain a single amino acid sequence.
- For **multimers**, the file should contain multiple sequences, one for each chain in the complex.

### 📁 Output (`--output`)
This directory will contain all AlphaFold outputs for the run, including:
- Final predicted structures in PDB and mmCIF formats.
- Confidence metrics (pLDDT scores and PAE matrices).
- Multiple sequence alignment (MSA) features.

### 🔄 Model Preset & Database Routing (`--preset`)
AlphaFold requires different background genetic tracking databases depending on the run mode. 

- **`monomer`**: Checks against the structural `pdb70` database.
- **`multimer`**: Completely bypasses the `pdb70` structural layer and evaluates across two separate databases: `pdb_seqres` and `uniprot`.

> [!TIP]
> You do not need to manually configure these database paths! The script automatically handles this database switch behind the scenes when you toggle `--preset multimer`.

## 🖥️ SLURM Resources

The script headers are currently configured to request the following HPC resources.

```bash
#SBATCH --partition=gpu
#SBATCH --gres=gpu:a100_3g40gb:1
#SBATCH --cpus-per-task=32
#SBATCH --mem=96G
```
- The script is run on one of the A100 GPU partitions on the cluster (`--partition=gpu` and --gres=gpu:a100_3g40gb:1). Please note AlphaFold can also run on a CPU, but it will be much much slower (~10x slower) compared to running it on a GPU. Also, if the GPU partition names are different on your machine, ensure to modify the `--gres` option to the appropriate argument. 
- It requests 32 CPUs on the host machine of the GPU partition (`--cpus-per-task=32`)
- Requests 96GB of memory (RAM) on the host machine (`--mem=96G`)


> [!NOTE]
> For more information about AlphaFold2 and the format of inputs required, please visit the [AlphaFold2 GitHub](https://github.com/google-deepmind/alphafold/)

> [!TIP]
> Before running the Slurm script on the ACE-Uganda HPC, make sure you have permission to run jobs on the GPU partition of the cluster. If not, or encounter any issues or have questions about configuring the SLURM resources, feel free to reach out to the support team at: support@ace-bioinformatics.org.


<div align="center">
  <h3>🧬 Happy Folding! 🧬</h3>
</div>


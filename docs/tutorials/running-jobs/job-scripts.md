---
id: job-scripts
title: Writing Job Scripts
---

SLURM job scripts are used to submit tasks to the cluster. These scripts define the resources required for the job, such as memory, time, and number of processors, as well as the software environment.

This guide will walk you through how to write a basic SLURM job script, using a HipSTR (short tandem repeat genotyping) job as an example.

---

## Basic Structure of a SLURM Job Script

A basic SLURM job script is a shell script that includes SLURM directives (prefixed with `#SBATCH`) to specify job parameters, followed by commands to run the actual task.

### HipSTR Job script

```bash
#!/bin/bash

#SBATCH --job-name=hipstr_job      # Name of the job
#SBATCH --output=hipstr_job.out    # Output file for standard output
#SBATCH --error=hipstr_job.err     # Output file for standard error
#SBATCH --time=12:00:00            # Maximum run time (HH:MM:SS)
#SBATCH --nodes=1                  # Number of nodes to allocate
#SBATCH --ntasks-per-node=8        # Number of tasks per node (set according to your job)
#SBATCH --mem=32gb                 # Memory allocation per node

# Load the necessary module (in this case, HipSTR)
module load HipSTR/0.6.2

# Run your job (example command for HipSTR)
HipSTR --bams samples.bam \
       --fasta reference.fa \
       --regions str_regions.bed \
       --str-vcf hipstr_results.vcf.gz \
       --log hipstr_run.log
```

---

### Explanation of SLURM Directives

- `#!/bin/bash`: Specifies the shell to use for running the script (bash in this case).
- `#SBATCH --job-name=hipstr_job`: Assigns a name to your job. This is helpful for identifying the job in the queue.
- `#SBATCH --output=hipstr_job.out`: Specifies the file where standard output (stdout) will be saved.
- `#SBATCH --error=hipstr_job.err`: Specifies the file where standard error (stderr) will be saved.
- `#SBATCH --time=12:00:00`: Specifies the maximum run time for the job in the format `HH:MM:SS`. Set this based on how long your task is expected to run.
- `#SBATCH --nodes=1`: Requests one compute node for the job. If your task needs more nodes, you can increase this number.
- `#SBATCH --ntasks-per-node=8`: Requests 8 tasks (processes) per node. This is typically set based on how many CPU cores your task needs.
- `#SBATCH --mem=32gb`: Specifies how much memory to allocate per node. If your task requires more memory, adjust this value.

---

## Loading Modules

Most HPC systems, including ACE, have software installed in modules. Before you run your job, you may need to load the required software module.

In the example above, `module load HipSTR/0.6.2` loads the specific version of HipSTR needed for the task.

You can check available modules on your system with:
```bash
module avail
```

---

## Writing the Commands to Run Your Task

After setting up your SLURM directives, you can include the commands to run your job. In the example, the job runs a `HipSTR` command:

```bash
HipSTR --bams samples.bam \
       --fasta reference.fa \
       --regions str_regions.bed \
       --str-vcf hipstr_results.vcf.gz \
       --log hipstr_run.log
```

This command performs short tandem repeat (STR) genotyping using the aligned reads in `samples.bam` against a reference genome (`reference.fa`), targeting the regions defined in `str_regions.bed`. The genotyped STR calls are written to `hipstr_results.vcf.gz` and a log of the run is saved to `hipstr_run.log`.

Key HipSTR arguments:
- `--bams`: One or more BAM files containing aligned reads (comma-separated for multiple samples).
- `--fasta`: The reference genome in FASTA format.
- `--regions`: A BED file specifying the STR regions to genotype.
- `--str-vcf`: Output VCF file with STR genotype calls.
- `--log`: Log file for run details and diagnostics.

---

## Submitting Your Job

Once your job script is ready, save it to a file, such as `hipstr_job.sh`. To submit the job to SLURM, use the `sbatch` command:

```bash
sbatch hipstr_job.sh
```

You'll receive a job ID upon submission, which you can use to track or cancel the job.

---

## Monitoring Your Job

After submission, you can monitor the status of your job using `squeue`:

```bash
squeue -u $USER
```

This will list all your active and queued jobs. If your job is running, you'll see its status here. You can also check detailed job information with `scontrol`:

```bash
scontrol show job <job_id>
```

---

## Checking Output and Errors

After the job completes, check the output and error files you specified (`hipstr_job.out` and `hipstr_job.err`). If the job completes successfully, you should find your results in the output file (`hipstr_results.vcf.gz`).

If your job fails, review the `.err` file to find details about what went wrong. Common issues could include incorrect file paths, missing dependencies, or resource allocation problems.

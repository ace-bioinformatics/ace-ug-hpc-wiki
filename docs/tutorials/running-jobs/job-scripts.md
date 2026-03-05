---
id: job-scripts
title: Writing Job Scripts
---

SLURM job scripts are used to submit tasks to the cluster. These scripts define the resources required for the job, such as memory, time, and number of processors, as well as the software environment.

This guide will walk you through how to write a basic SLURM job script, using a BLAST (bioinformatics) job as an example.

---

## Basic Structure of a SLURM Job Script

A basic SLURM job script is a shell script that includes SLURM directives (prefixed with `#SBATCH`) to specify job parameters, followed by commands to run the actual task.

### Example SLURM Job Script: BLAST Job

```bash
#!/bin/bash

#SBATCH --job-name=blast_job       # Name of the job
#SBATCH --output=blast_job.out     # Output file for standard output
#SBATCH --error=blast_job.err      # Output file for standard error
#SBATCH --time=12:00:00            # Maximum run time (HH:MM:SS)
#SBATCH --nodes=1                  # Number of nodes to allocate
#SBATCH --ntasks-per-node=8        # Number of tasks per node (set according to your job)
#SBATCH --mem=32gb                 # Memory allocation per node

# Load the necessary module (in this case, BLAST)
module load blast/2.12.0

# Run your job (example command for BLAST)
blastn -query input.fasta -db database -out results.txt -num_threads 8
```

---

### Explanation of SLURM Directives

- `#!/bin/bash`: Specifies the shell to use for running the script (bash in this case).
- `#SBATCH --job-name=blast_job`: Assigns a name to your job. This is helpful for identifying the job in the queue.
- `#SBATCH --output=blast_job.out`: Specifies the file where standard output (stdout) will be saved.
- `#SBATCH --error=blast_job.err`: Specifies the file where standard error (stderr) will be saved.
- `#SBATCH --time=12:00:00`: Specifies the maximum run time for the job in the format `HH:MM:SS`. Set this based on how long your task is expected to run.
- `#SBATCH --nodes=1`: Requests one compute node for the job. If your task needs more nodes, you can increase this number.
- `#SBATCH --ntasks-per-node=8`: Requests 8 tasks (processes) per node. This is typically set based on how many CPU cores your task needs.
- `#SBATCH --mem=32gb`: Specifies how much memory to allocate per node. If your task requires more memory, adjust this value.

---

## Loading Modules

Most HPC systems, including ACE, have software installed in modules. Before you run your job, you may need to load the required software module.

In the example above, `module load blast/2.12.0` loads the specific version of the BLAST software needed for the task.

You can check available modules on your system with:
```bash
module avail
```

---

## Writing the Commands to Run Your Task

After setting up your SLURM directives, you can include the commands to run your job. In the example, the job runs a `blastn` command:

```bash
blastn -query input.fasta -db database -out results.txt -num_threads 8
```

This command performs a BLAST search with the input file `input.fasta` against the `database`, saving the results to `results.txt`. The `-num_threads 8` argument tells BLAST to use 8 threads, corresponding to the number of tasks requested in the SLURM job script.

---

## Submitting Your Job

Once your job script is ready, save it to a file, such as `blast_job.sh`. To submit the job to SLURM, use the `sbatch` command:

```bash
sbatch blast_job.sh
```

You’ll receive a job ID upon submission, which you can use to track or cancel the job.

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

After the job completes, check the output and error files you specified (`blast_job.out` and `blast_job.err`). If the job completes successfully, you should find your results in the output file (`results.txt`).

If your job fails, review the `.err` file to find details about what went wrong. Common issues could include incorrect file paths, missing dependencies, or resource allocation problems.

---

## Best Practices for Writing SLURM Scripts

- **Resource Allocation**: Always request the appropriate resources. If your job uses too many or too few, it may fail to start or cause inefficient resource usage. Adjust `--mem`, `--time`, `--nodes`, and `--ntasks-per-node` according to your task’s requirements.
  
- **Modularize Your Script**: If your script requires multiple dependencies, load them in sequence using `module load`.

- **Use Relative Paths**: Use relative paths for input/output files (e.g., `input.fasta`) unless your job requires absolute paths.

- **Test Before Full Run**: For large or long-running jobs, try to test your script with a smaller dataset or a shorter run time to ensure everything works as expected.

---

## Example: Another Bioinformatics Task (Hisat2)

Here’s an example of a job script for a HISAT2 task:

```bash
#!/bin/bash
#SBATCH --job-name=hisat2_job
#SBATCH --output=hisat2_job.out
#SBATCH --error=hisat2_job.err
#SBATCH --time=02:00:00
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=8
#SBATCH --mem=16gb

module load hisat2/2.2.1

hisat2 -p 8 -x genome_index -U reads.fastq -S output.sam
```

In this example:
- `-p 8`: Uses 8 threads.
- `-x genome_index`: Specifies the genome index for alignment.
- `-U reads.fastq`: Input file for single-end reads.
- `-S output.sam`: Output SAM file.

---

## Troubleshooting Tips

- **Job won't start**: Check if there are sufficient resources available with `squeue` or `sinfo`. If needed, reduce resource requests like `--time` or `--mem`.
- **Job fails with segmentation fault**: Test the job interactively (`salloc`) to debug any code issues in a smaller test.
- **Incorrect output**: Ensure all file paths are correct and that you’re using the correct module versions.

---

## Conclusion

Writing and submitting a SLURM job script is the standard way to run tasks on the ACE HPC Cluster. By setting appropriate resource limits, loading necessary software modules, and properly writing the commands to run your tasks, you can efficiently use the cluster’s resources.

For more examples and advanced topics, refer to the [SLURM documentation](https://slurm.schedmd.com/documentation.html).
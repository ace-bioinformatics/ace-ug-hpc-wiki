---
id: sample-scripts
title: Sample SLURM Job Scripts
---

This page provides a collection of sample SLURM job scripts designed for various purposes. These scripts are intended for demonstration purposes and can be used to understand the structure of SLURM job submissions and how to interact with the cluster.

---

## Sample Script 1: Printing Loaded Modules

This script lists all currently loaded modules on the cluster, which can help you understand which software packages are available for use.

### Script: `list_modules.sh`

```bash
#!/bin/bash
#SBATCH --job-name=list_modules       # Job name
#SBATCH --output=list_modules.out     # Standard output file
#SBATCH --error=list_modules.err      # Standard error file

# List currently loaded modules
module list
```

### How to Use:
- Save this script as `list_modules.sh`.
- Submit the script using `sbatch list_modules.sh`.
- Check the output in the `list_modules.out` file.

---

## Sample Script 2: Checking Available Disk Space

This script prints the available disk space on the system, which can be useful to monitor the storage usage on your home or project directories.

### Script: `check_disk_space.sh`

```bash
#!/bin/bash
#SBATCH --job-name=check_disk_space    # Job name
#SBATCH --output=check_disk_space.out  # Standard output file
#SBATCH --error=check_disk_space.err   # Standard error file

# Check available disk space
df -h
```

### How to Use:
- Save this script as `check_disk_space.sh`.
- Submit the script using `sbatch check_disk_space.sh`.
- The available disk space information will be output to `check_disk_space.out`.

---

## Sample Script 3: Printing the Date and Time

This script prints the current date and time, which can help you verify when your job was started and track the duration of the job.

### Script: `print_date.sh`

```bash
#!/bin/bash
#SBATCH --job-name=print_date       # Job name
#SBATCH --output=print_date.out     # Standard output file
#SBATCH --error=print_date.err      # Standard error file

# Print the current date and time
date
```

### How to Use:
- Save this script as `print_date.sh`.
- Submit the script using `sbatch print_date.sh`.
- The current date and time will be printed in the `print_date.out` file.

---

## Sample Script 4: Running a Simple Python Script

This script runs a simple Python script that prints "Hello, World!", which demonstrates how to submit and run a Python job using SLURM.

### Script: `run_python_demo.sh`

```bash
#!/bin/bash
#SBATCH --job-name=python_demo       # Job name
#SBATCH --output=python_demo.out     # Standard output file
#SBATCH --error=python_demo.err      # Standard error file
#SBATCH --mem=1gb                    # Memory request

# Load Python module (if needed)
module load python/3.8

# Run a simple Python script that prints "Hello, World!"
echo "print('Hello, World!')" | python
```

### How to Use:
- Save this script as `run_python_demo.sh`.
- Submit the script using `sbatch run_python_demo.sh`.
- The output will be shown in `python_demo.out`, where it should print "Hello, World!".

---

## Sample Script 5: Running a Simple Shell Command (Echo)

This script demonstrates how to use a simple `echo` command to print a message, which is useful for testing job submission with minimal complexity.

### Script: `echo_message.sh`

```bash
#!/bin/bash
#SBATCH --job-name=echo_message       # Job name
#SBATCH --output=echo_message.out     # Standard output file
#SBATCH --error=echo_message.err      # Standard error file

# Print a custom message to the output file
echo "This is a demo job running on the cluster!"
```

### How to Use:
- Save this script as `echo_message.sh`.
- Submit the script using `sbatch echo_message.sh`.
- The message "This is a demo job running on the cluster!" will appear in the `echo_message.out` file.

---

## Conclusion

These simple SLURM job scripts provide basic examples for running commands, checking system status, and executing scripts. By following these examples, even beginners can start submitting jobs to the cluster and experiment with SLURM job scripts. 

Feel free to modify these scripts to suit your needs and explore further SLURM features! For more advanced scripting and options, refer to the [SLURM documentation](https://slurm.schedmd.com/documentation.html).
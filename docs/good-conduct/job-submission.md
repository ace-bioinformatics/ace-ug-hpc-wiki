# Job Submission Tips

Submitting jobs responsibly ensures fair access to compute resources for all ACE users. Request only what you need, test before scaling up, and monitor your jobs to understand their resource usage.

## Request Only What You Need

Over-requesting resources wastes cluster capacity and increases your queue wait time.

### Nodes and Tasks

```bash
# BAD: Requesting more resources than needed
#SBATCH --nodes=10
#SBATCH --ntasks-per-node=32
# ... but your code only uses 1 node

# GOOD: Match resources to your actual needs
#SBATCH --nodes=1
#SBATCH --ntasks=8
```

### Time Limits

Request realistic time limits:

```bash
# BAD: Always requesting maximum time
#SBATCH --time=7-00:00:00  # 7 days "just in case"

# GOOD: Based on tested runtime + buffer
#SBATCH --time=04:00:00  # 4 hours (tested: 3 hours + 1 hour buffer)
```

**Why this matters:**
- Shorter jobs can backfill into gaps, starting sooner
- Over-requested time blocks resources others could use
- The scheduler prioritizes jobs that fit available windows

### Memory

```bash
# Specify memory only if you know your requirements
#SBATCH --mem=16G          # Total memory per node
#SBATCH --mem-per-cpu=4G   # Memory per CPU core
```

## Test Before Scaling

Always verify your job works at small scale before requesting large resources.

### Interactive Testing

```bash
# Start an interactive session
srun --nodes=1 --ntasks=1 --time=00:30:00 --pty bash

# Test your code runs
./my_program --test-mode
```

### Small Batch Test

```bash
# Submit a minimal test job
#SBATCH --nodes=1
#SBATCH --ntasks=2
#SBATCH --time=00:30:00

./my_program --small-input
```

### Scale Up Gradually

```bash
# After small tests succeed, gradually increase
# 1 node → 2 nodes → 4 nodes → target size

# Monitor each step before proceeding
```

## Monitor Resource Usage

Understanding how your jobs use resources helps you request appropriately.

### During Job Execution

```bash
# Check running jobs
squeue -u $USER

# Get detailed job info
scontrol show job <jobid>

# SSH to compute node and check resources (if allowed)
ssh <nodename>
top -u $USER
```

### After Job Completion

```bash
# View job efficiency
seff <jobid>

# Example output:
# Job ID: 12345
# CPU Efficiency: 85.2%
# Memory Efficiency: 62.3% of 16.00 GB
```

If efficiency is low, reduce your resource requests.

### Using sacct

```bash
# View completed job statistics
sacct -j <jobid> --format=JobID,Elapsed,MaxRSS,MaxVMSize,CPUTime

# View your recent jobs
sacct -u $USER --starttime=2024-01-01 --format=JobID,JobName,Elapsed,State
```

## Job Array Best Practices

Job arrays are efficient for parameter sweeps but require care:

```bash
# GOOD: Reasonable array size with throttling
#SBATCH --array=1-100%10  # 100 tasks, max 10 running at once

# BAD: Thousands of tiny tasks flooding the scheduler
#SBATCH --array=1-10000   # No throttling
```

### Throttle Large Arrays

```bash
# Limit concurrent tasks
#SBATCH --array=1-1000%50  # Max 50 running simultaneously
```

### Combine Small Tasks

If each task runs for only seconds, combine them:

```bash
# Instead of 10,000 one-second tasks
# Create 100 tasks that each process 100 items
```

## Avoid Common Mistakes

### Don't Flood the Queue

```bash
# BAD: Submitting thousands of jobs in a loop
for i in {1..5000}; do
    sbatch job_$i.sh
done

# GOOD: Use job arrays
#SBATCH --array=1-5000%100
```

### Don't Ignore Failed Jobs

```bash
# Check for failures
sacct -u $USER --state=FAILED --starttime=2024-01-01

# Investigate before resubmitting
cat slurm-<jobid>.out
```

### Don't Hardcode Paths

```bash
# BAD: Hardcoded paths that break when moved
cd /home/username/project
./run.sh

# GOOD: Use environment variables
cd $SLURM_SUBMIT_DIR
./run.sh
```

## Dependency Chains

Use job dependencies instead of sleep loops:

```bash
# Submit first job
JOB1=$(sbatch --parsable preprocess.sh)

# Submit dependent job
sbatch --dependency=afterok:$JOB1 analysis.sh
```

## Summary Checklist

Before submitting, verify:

- [ ] Resources match tested requirements
- [ ] Time limit based on actual runtime
- [ ] Job tested at small scale first
- [ ] Array jobs are throttled appropriately
- [ ] Output directory exists and has space
- [ ] No hardcoded paths

| Practice | Impact |
|----------|--------|
| Right-size resources | Faster queue times, fair sharing |
| Test before scaling | Avoid wasted allocations |
| Monitor efficiency | Improve future requests |
| Throttle arrays | Reduce scheduler load |
| Use dependencies | Clean workflow management |

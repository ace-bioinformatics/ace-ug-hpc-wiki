---
id: contact
title: Getting Help
---

# Getting Help

ACE Uganda support staff are here to help you with account issues, software questions, job failures, and general HPC guidance. Before submitting a support request, take a moment to review the tips below — a well-written request helps us resolve your issue faster.

## How to Contact Us

| Channel | Details |
|---------|---------|
| **Email** | [support@ace-bioinformatics.org](mailto:support@ace-bioinformatics.org) |
| **Hours** | Monday – Friday, 9:00 AM – 5:00 PM EAT (excluding public holidays) |
| **Response time** | Typically within 1–2 business days |

## Before You Submit a Request

Do some initial troubleshooting on your own — it's often the fastest path to a solution:

- **Check the documentation.** Browse the [FAQs](faq.md), [Good Conduct](../good-conduct/overview.md) guidelines, and [Tutorials](../tutorials/overview.md) sections of this wiki.
- **Read the error message carefully.** Most error messages describe exactly what went wrong. Search the message text online if it's unfamiliar.
- **Check your job output files.** Review `.out` and `.err` files from your job for clues. Run `sacct -j <jobid> --format=JobID,State,ExitCode,MaxRSS` for resource usage details.
- **Verify your environment.** Run `module list` to confirm the right modules are loaded, and check that file paths in your scripts are correct.

## Writing an Effective Support Request

A clear, detailed request allows us to diagnose your issue without multiple back-and-forth emails. Include the following:

### What to include

1. **Your username** on the cluster.
2. **What you were trying to do** — a brief description of your goal.
3. **What you did** — the exact commands or steps you ran.
4. **What happened** — paste the **complete, verbatim error message** or unexpected output. Do not paraphrase errors.
5. **What you expected** to happen instead.
6. **Job details** (if applicable):
   - Job ID(s): find with `sacct -u $USER`
   - The directory containing your job script and data
   - Your job script (attach or paste it in full)
   - Any loaded modules (`module list`)
7. **What you already tried** to fix the issue.

### Example of a good support request

> **Subject:** Job 485210 — out-of-memory error running BLAST
>
> Hi, I'm user `jdoe`. I submitted a BLAST job (ID 485210) in `/home/jdoe/blast_project/` using the script `run_blast.sh`. The job was killed after 20 minutes with the error:
>
> `slurmstepd: error: Detected 1 oom_kill event in StepId=485210.0. Some of the step tasks have been OOM Killed.`
>
> I requested 4 GB of memory with `--mem=4G`. My input database is 12 GB. I suspect I need more memory but I'm not sure how much to request. I tried 8 GB and it still failed. Could you advise?

### Example of a poor support request

> **Subject:** Job not working
>
> My job failed. Can you help?

The more context you provide upfront, the faster we can help.

## What We Can Help With

- **Account issues** — password resets, access problems, quota increases
- **Job failures** — debugging Slurm errors, resource allocation issues, out-of-memory kills
- **Software** — questions about installed software, module loading, and custom installations
- **Data management** — file transfers, storage quotas, data organization
- **General HPC guidance** — choosing the right resources for your workload, parallelization strategies

## What Falls Outside Our Scope

Our team focuses on cluster operations and general HPC guidance. Some requests are better addressed elsewhere:

- **Debugging your application code** — we can help with HPC-specific issues (MPI errors, Slurm failures), but we cannot debug the logic of your research code.
- **Teaching programming languages** — we recommend online resources, workshops, or your department's training programs for learning Python, R, etc.
- **Third-party software internals** — we can help install and configure software, but for application-specific scientific questions, consult the software's own documentation or community forums.

## Stay Informed

- **System announcements** — maintenance windows, outages, and updates are communicated via email to all registered users. Make sure your contact email is up to date.
- **Documentation updates** — this wiki is regularly updated. Check back for new tutorials and guides.

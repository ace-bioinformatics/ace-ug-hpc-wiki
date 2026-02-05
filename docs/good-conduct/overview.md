# Good Conduct on ACE HPC

As a shared resource, ACE HPC serves many researchers simultaneously. Your actions directly impact the experience of other users. Following these guidelines ensures equitable access and optimal performance for everyone.

## Core Principles

1. **Do not run jobs on login nodes** - Login nodes are shared gateways, not compute resources
2. **Respect file system quotas** - Exceeding quotas affects your jobs and strains shared storage
3. **Submit jobs responsibly** - Request only the resources you need
4. **Transfer files efficiently** - Use appropriate methods to avoid overloading the network

## Quick Reference

| Do | Don't |
|---|---|
| Submit jobs via SLURM | Run computations on login nodes |
| Monitor your quota usage | Exceed allocated storage |
| Use `$SCRATCH` for job I/O | Run I/O-intensive jobs in `$HOME` |
| Archive small files before transfer | Transfer thousands of tiny files |
| Request appropriate resources | Over-request nodes or time |

## Why This Matters

The ACE cluster is a shared community resource. When one user monopolizes login nodes or overwhelms the file system, it degrades performance for everyone. Responsible conduct ensures:

- **Fair access** - All researchers can work effectively
- **System stability** - Infrastructure remains healthy and responsive
- **Faster job turnaround** - Resources are used efficiently

## Consequences of Misuse

Repeated violations of good conduct guidelines may result in:

- Account suspension
- Reduced resource allocations
- Required consultation with support staff before reinstatement

## Getting Help

If you're unsure whether your workflow follows good conduct guidelines, contact the ACE support team **before** running large jobs. We're here to help you succeed while maintaining a healthy shared environment.

- Email: [support@ace.ac.ug](mailto:support@ace.ac.ug)
- See the [FAQ](/docs/support/faq) for common questions

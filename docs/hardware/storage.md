---
id: storage
title: Storage Systems
---

The ACE HPC Cluster provides the following storage systems.

### File Systems
| Name      | Path             | Size   | Quota                        | Purpose              |
|-----------|------------------|--------|------------------------------|----------------------|
| Home      | `/home/username` | —      | 50 GB default; up to 200 GB  | User home directory  |
| Root      | `/`              | 180 GB | System use                   | Operating system     |
| ace-data1 | `/etc/ace-data`  | 190 TB | Shared (project directories) | Large shared storage |

### Quotas

**User home directories** — each user is allocated 50 GB by default. Storage can be increased up to 200 GB upon submitting a valid request and justification to [support@ace-bioinformatics.org](mailto:support@ace-bioinformatics.org).

**Project directories** — each project is allocated 1 TB by default. Storage can be increased up to 5 TB with a valid request and justification.

### Policies
- **Home (`$HOME`)**: Your primary workspace. Used for scripts, configs, software, datasets, and job output. Daily backups retained for 90 days.
- **ace-data1**: Shared space for project directories; manage usage responsibly. Project directories are **not** automatically backed up — project owners are responsible for backing up critical data.
- **Root**: Reserved for system files — do not store data here.

### Usage Tips
- `$HOME` is your primary workspace. Organise it with subdirectories (`projects/`, `jobs/`, `software/`).
- Request a project directory on ace-data1 for large collaborative datasets that exceed your personal quota.
- Monitor your usage with `quota -s` — you will be alerted when you reach 90% of your quota.

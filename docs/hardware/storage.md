---
id: storage
title: Storage Systems
---

The ACE HPC Cluster provides multiple storage options for users.

### File Systems
| Name      | Path             | Size  | Quota       | Purpose              |
|-----------|------------------|-------|-------------|----------------------|
| Root      | `/`              | 180 GB| System use  | Operating system     |
| ace-data1 | `/etc/ace-data`  | 190 TB| Shared      | Large shared storage |

### Policies
- **ace-data1**: Shared space; manage usage responsibly.
- **Root**: Reserved for system files—do not store data here.

### Usage Tips
- Use ace-data1 for large datasets and shared project files.
- Keep $HOME for lightweight, personal scripts or configs.

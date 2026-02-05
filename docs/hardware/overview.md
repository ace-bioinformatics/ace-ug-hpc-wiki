---
id: overview
title: Hardware Overview
---

The ACE HPC Cluster is designed for bioinformatics and data-intensive research. Below are its key components.

### Compute Nodes
| Type         | Quantity | Cores    | Memory    | Storage       |
|--------------|----------|----------|-----------|---------------|
| Standard     | 56       | 15,232   |  5,376 GB |  503.2 TB     |

## Specifications
Each of the nodes is a Dell PowerEdge C6320p, a high density, 2U rack server designed for High Performance Computing (HPC) environments.

### CPU

| Property                | Value                                 |
|-------------------------|----------------------------------------|
| Architecture            | x86_64                                 |
| CPU op-mode(s)          | 32-bit, 64-bit                         |
| Address sizes           | 46 bits physical, 48 bits virtual     |
| Byte Order              | Little Endian                         |
| CPU(s)                  | 272                                    |
| On-line CPU(s) list     | 0-271                                  |
| Vendor ID               | GenuineIntel                          |
| Model name              | Intel(R) Xeon Phi(TM) CPU 7250 @ 1.40GHz |
| CPU family              | 6                                     |
| Model                   | 87                                    |
| Thread(s) per core      | 4                                     |
| Core(s) per socket      | 68                                     |
| Memory                  | 96 GB                                  |
| Socket(s)               | 1                                     |
| Stepping                | 1                                     |

## Interconnect
- 10 Gb/s NREN connection (RENU).

## Login Nodes
- 1 login node, with 64 cores and 16 GB RAM.

## Additional Details
- 200Mbps internet connection (RENU).

See [Storage](storage) for file system details.
---
id: overview
title: Hardware Overview
---

The ACE HPC Cluster is designed for bioinformatics and data-intensive research. Below are its key components.

### CPU Compute Nodes
| Type         | Quantity | Total Cores    | Total Memory    | Storage       |
|--------------|----------|----------------|-----------------|---------------|
| CPU          | 56       | 15,232         |  5,376 GB       |  503.2 TB     |

### GPU Compute Nodes
| Type         | Quantity | CUDA Cores    | Memory    | Interconnect      |
|--------------|----------|---------------|-----------|---------------|
| GPU          | 4        | 27,648        |  320 GB   |  NVLink (600 GB/s) or PCIe Gen 4 (64 GB/s)     |

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

### GPU
| Property                   | Value                                  |
|----------------------------|----------------------------------------|
| Architecture               | NVIDIA Ampere                          |
| GPU Variant                | GA100-880-A1                           |
| Process Size               | 7nm (TSMC)                             |
| Transistors                | 54.2 Billion                           |
| CUDA Cores                 | 6,912                                  |
| Tensor Cores               | 	432 (3rd Generation)                  |
| Memory Size                | 80GB                                   |
| Memory Type                | HBM2e                                  |
| Memory Bus Width           | 5,120-bit                              |
| Memory Bandwidth           | 19.5 TFLOPS                            |
| FP32 Performance           | 9.7 TFLOPS                             |
| TF32 Tensor Core           | 156 TFLOPS (312 TFLOPS Sparse)         |
| BFLOAT16 Tensor Core       | 312 TFLOPS (624 TFLOPS Sparse)         |
| Multi-Instance GPU         | Up to 7 Instances (10GB each)          |
| Interconnect               | NVLink (600 GB/s) / PCIe 4.0 x16       |
| TDP (Thermal Design Power) | 400W (SXM) / 300W (PCIe)               |

## Interconnect
- 10 Gb/s NREN connection (RENU).

## Login Nodes
- 1 login node, with 64 cores and 16 GB RAM.

## Additional Details
- 200Mbps internet connection (RENU).

See [Storage](storage) for file system details.
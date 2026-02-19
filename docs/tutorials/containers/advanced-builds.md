---
id: advanced-builds
title: Advanced Build Topics
sidebar_label: Advanced Build Topics
sidebar_position: 3
---

# Advanced Build Topics

This tutorial covers two techniques that become important as you move beyond basic Dockerfiles: **multi-stage builds** for producing smaller images and **multi-architecture builds** for supporting different CPU architectures.

## Multi-Stage Builds

### The Problem

When you build a container that compiles code or installs packages from source, the build tools (compilers, headers, development libraries) end up in the final image even though they're only needed during the build. This bloats your image.

Consider our Monte Carlo example from the previous tutorial. The `python:3.11-slim` base image is ~125 MB, but after adding `build-essential` for compiling numpy, the image grows significantly. Those compilers serve no purpose at runtime — they just waste disk space and increase pull times on the cluster.

### The Concept

Multi-stage builds (introduced in Docker 17.06) solve this by letting you use multiple `FROM` statements in a single Dockerfile. Each `FROM` begins a new build stage. You can copy artifacts from earlier stages into later ones, leaving the build tools behind.

```
┌──────────────────────┐       ┌──────────────────────┐
│    Stage 1: builder  │       │   Stage 2: runtime   │
│                      │       │                      │
│  Base: python:3.11   │       │  Base: python:3.11-  │
│  + build-essential   │ COPY  │        slim           │
│  + gfortran          │ ───>  │  + /opt/venv (from   │
│  + libopenblas-dev   │       │    builder stage)     │
│  + /opt/venv with    │       │  + app code           │
│    compiled packages │       │                      │
│                      │       │  No compilers!        │
│  Size: ~1.5 GB       │       │  Size: ~300 MB        │
└──────────────────────┘       └──────────────────────┘
```

The key instruction is `COPY --from=<stage>`, which copies files from a named stage instead of from the build context.

### The Example

Let's rebuild our Monte Carlo simulation with a multi-stage approach. We'll add `scipy` to the requirements so there's a meaningful amount of compiled code to separate.

Create the project:

```bash
mkdir ~/monte-carlo-multistage && cd ~/monte-carlo-multistage
```

Create `requirements.txt`:

```
numpy==1.24.3
scipy==1.11.1
```

Create `estimate_pi.py` (same as before, but now using scipy for a comparison):

```python
#!/usr/bin/env python3
"""Estimate Pi using Monte Carlo and scipy's statistical tools."""

import argparse
import numpy as np
from scipy import stats
import time
import json

def monte_carlo_pi(num_samples):
    """Estimate Pi via random sampling."""
    x = np.random.uniform(0, 1, num_samples)
    y = np.random.uniform(0, 1, num_samples)
    inside = np.sum(x**2 + y**2 <= 1)
    return 4 * inside / num_samples

def main():
    parser = argparse.ArgumentParser(description="Monte Carlo Pi estimation with statistics")
    parser.add_argument("samples", type=int, nargs="?", default=1_000_000)
    parser.add_argument("--trials", type=int, default=10,
                        help="Number of independent trials")
    parser.add_argument("--output", "-o", type=str, default=None)
    args = parser.parse_args()

    print(f"Running {args.trials} trials of {args.samples:,} samples each...")
    start = time.time()

    estimates = [monte_carlo_pi(args.samples) for _ in range(args.trials)]
    elapsed = time.time() - start

    mean = np.mean(estimates)
    ci = stats.t.interval(0.95, df=len(estimates)-1,
                          loc=mean, scale=stats.sem(estimates))

    print(f"  Mean estimate: {mean:.8f}")
    print(f"  95% CI:        [{ci[0]:.8f}, {ci[1]:.8f}]")
    print(f"  Actual Pi:     {np.pi:.8f}")
    print(f"  Total time:    {elapsed:.3f}s")

    if args.output:
        results = {
            "trials": args.trials,
            "samples_per_trial": args.samples,
            "mean": mean,
            "ci_lower": ci[0],
            "ci_upper": ci[1],
            "elapsed_seconds": elapsed,
        }
        with open(args.output, "w") as f:
            json.dump(results, f, indent=2)
        print(f"  Results written to {args.output}")

if __name__ == "__main__":
    main()
```

Now create a `Dockerfile` with two stages:

```dockerfile
# ---- Stage 1: Builder ----
# This stage installs compilers and builds all Python packages.
FROM python:3.11 AS builder

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        gfortran \
        libopenblas-dev \
    && rm -rf /var/lib/apt/lists/*

# Install packages into a virtual environment so we can copy it cleanly.
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt


# ---- Stage 2: Runtime ----
# This stage starts fresh and copies only the compiled packages.
FROM python:3.11-slim

# Install only the runtime libraries (no compilers, no headers).
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        libopenblas0 \
        libgomp1 \
    && rm -rf /var/lib/apt/lists/*

# Copy the virtual environment from the builder stage.
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

WORKDIR /app
COPY estimate_pi.py .
RUN chmod +x estimate_pi.py

CMD ["python", "estimate_pi.py"]
```

**What's happening here:**

- **Stage 1** (`AS builder`) starts from the full `python:3.11` image (~1 GB), installs compilers and development headers, creates a virtual environment, and compiles numpy and scipy inside it. This stage is a build tool — it produces `/opt/venv` with all the compiled packages.

- **Stage 2** starts fresh from `python:3.11-slim` (~125 MB). It installs only the *runtime* libraries (the shared `.so` files that numpy/scipy link against, without the headers or compilers). Then `COPY --from=builder /opt/venv /opt/venv` pulls the compiled virtual environment from stage 1 into the clean image.

The compilers, development headers, and all of stage 1's baggage are left behind. They exist during the build but are not in the final image.

Build both versions and compare:

```bash
# Multi-stage build
docker build -t monte-carlo-ms:0.1 .

# For comparison, build a single-stage version
cat > Dockerfile.single << 'EOF'
FROM python:3.11
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential gfortran libopenblas-dev \
    && rm -rf /var/lib/apt/lists/*
RUN pip install --no-cache-dir numpy==1.24.3 scipy==1.11.1
WORKDIR /app
COPY estimate_pi.py .
CMD ["python", "estimate_pi.py"]
EOF

docker build -f Dockerfile.single -t monte-carlo-single:0.1 .

# Compare image sizes
docker images | grep monte-carlo
```

```
monte-carlo-single   0.1    ...   1.41GB
monte-carlo-ms       0.1    ...   327MB
```

The multi-stage image is roughly **4x smaller** with identical functionality.

<!-- TODO: Add a screenshot showing the docker images output comparing the two sizes -->

Test that the multi-stage image works:

```bash
docker run --rm monte-carlo-ms:0.1 python estimate_pi.py 1000000 --trials 5
```

:::tip Debugging build stages
You can build up to a specific stage using `--target`:
```bash
docker build --target builder -t monte-carlo-debug:0.1 .
docker run --rm -it monte-carlo-debug:0.1 /bin/bash
```
This gives you a shell in the builder stage so you can inspect what was installed, check library paths, and troubleshoot compilation issues.
:::

---

## Multi-Architecture Builds

### The Problem

Container images are built for a specific CPU architecture. An image built on an Intel/AMD laptop (linux/amd64) won't run on an ARM-based Mac (linux/arm64), or on other architectures found in some HPC clusters.

If you build on a Mac with Apple Silicon, the image defaults to `linux/arm64`. If ACE HPC nodes use `linux/amd64`, the container either fails or runs under slow emulation.

### The Concept

Multi-architecture builds create a single image tag that contains variants for multiple architectures. When someone pulls the image, Docker or Apptainer automatically selects the correct variant for their platform. This is handled by `docker buildx`, which uses QEMU emulation to build for architectures other than your host.

```
                     yourusername/monte-carlo:0.2
                              │
               ┌──────────────┼──────────────┐
               │              │              │
          linux/amd64    linux/arm64    linux/ppc64le
           (Intel)     (Apple M-series)   (POWER)
```

### Prerequisites

- **Docker Desktop** (Mac/Windows): Includes `buildx` and QEMU emulation out of the box.
- **Docker Engine on Linux**: You may need to install QEMU separately:
  ```bash
  docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
  ```

### The Example

We'll take the same Monte Carlo simulation and build it for both `linux/amd64` and `linux/arm64`.

First, create a new builder instance that supports multi-platform builds:

```bash
# List existing builders
docker buildx ls

# Create a new builder with multi-platform support
docker buildx create --name multiplatform --bootstrap --use
```

<!-- TODO: Add a screenshot showing the output of `docker buildx ls` with the new builder -->

Now build for a single target platform to test:

```bash
cd ~/monte-carlo-multistage

# Build for amd64 only and load into local Docker
docker buildx build --platform linux/amd64 -t monte-carlo-ms:0.2-amd64 --load .

# Verify the architecture
docker inspect monte-carlo-ms:0.2-amd64 | grep Architecture
```

```json
"Architecture": "amd64"
```

Once you've confirmed the single-platform build works, build for multiple architectures. Multi-architecture images **must be pushed directly to a registry** — they cannot be loaded into local Docker because the local daemon only supports one architecture at a time:

```bash
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    -t yourusername/monte-carlo:0.2 \
    --push \
    .
```

This builds the Dockerfile twice — once under `linux/amd64` and once under `linux/arm64` (using QEMU emulation for the non-native architecture). Both images are pushed to Docker Hub under the same tag. When someone pulls `yourusername/monte-carlo:0.2`, Docker automatically selects the variant matching their platform.

Verify the manifest:

```bash
docker buildx imagetools inspect yourusername/monte-carlo:0.2
```

This shows the available platforms under that tag.

### Considerations

**Build time:** Building for a non-native architecture is slower because it runs under QEMU emulation. A build that takes 2 minutes natively may take 10–15 minutes under emulation.

**Base image support:** Your base image must support all the target architectures. Most official images (`python`, `ubuntu`, `alpine`) support both `amd64` and `arm64`. Check the image's page on Docker Hub to see its supported architectures.

**Apptainer on ACE HPC:** When you pull a multi-arch image with Apptainer, it automatically selects the architecture matching the cluster's hardware:

```bash
module load apptainer
apptainer pull docker://yourusername/monte-carlo:0.2
# Automatically pulls the linux/amd64 variant on x86_64 nodes
```

### Automating with GitHub Actions

For production workflows, you can automate multi-architecture builds with GitHub Actions so that every push to your repository builds and pushes images for all platforms:

```yaml
name: Build and Push Container

on:
  push:
    tags:
      - "v*"

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-qemu-action@v3

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/monte-carlo:${{ github.ref_name }}
```

This workflow triggers on version tags (e.g., `git tag v0.3 && git push --tags`). It sets up QEMU and buildx, logs into Docker Hub using secrets stored in the repository, and builds + pushes for both architectures.

Store your Docker Hub credentials as GitHub repository secrets (`DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN`) under **Settings > Secrets and variables > Actions**.

---

**Next:** [Containers on HPC Clusters](containers-hpc) — pull your images on ACE HPC with Apptainer, write SLURM job scripts, and run MPI and GPU workloads.

**References:** [Docker Multi-stage Builds](https://docs.docker.com/build/building/multi-stage/) | [Docker Buildx](https://docs.docker.com/build/building/multi-platform/) | [QEMU User Emulation](https://github.com/multiarch/qemu-user-static)

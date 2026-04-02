# GROMACS Docker Image

**Location**: `docker/gromacs/Dockerfile`  
**Published to**: `ghcr.io/jinzhanglab/gmxpytools/gromacs`

---

## Overview

A multi-stage Docker image that compiles [GROMACS](https://www.gromacs.org/) from source and ships a lean runtime image with:

| Binary  | Precision | GPU support | Use case |
|---------|-----------|-------------|----------|
| `gmx`   | Single    | ✅ CUDA     | Production MD runs, free energy, etc. |
| `gmx_d` | Double    | ❌ CPU only | High-accuracy calculations (normal mode analysis, energy minimisation) |

> **Why no GPU for double precision?**  
> NVIDIA CUDA does not support double-precision MD kernels. `gmx_d` is CPU-only by design.

---

## Image design

### Multi-stage build

```
Stage 1 (builder): nvidia/cuda:<ver>-devel-ubuntu22.04
  └─ Installs build tools, compilers, CMake, OpenMPI dev
  └─ Compiles gmx  (single-precision, CUDA)
  └─ Compiles gmx_d (double-precision, CPU)

Stage 2 (final):  nvidia/cuda:<ver>-runtime-ubuntu22.04   ← much smaller
  └─ Only runtime libs: libopenmpi3, libgomp1, CUDA runtime + cuFFT
  └─ Copies /usr/local/gromacs from Stage 1
```

The `runtime` CUDA image already includes `libcudart` and cuFFT (math libraries). The CUDA driver itself is provided by the host via **nvidia-container-toolkit**; it is never baked into the image.

FFTW is built by GROMACS itself (`GMX_BUILD_OWN_FFTW=ON`) and statically linked, so no `libfftw3` package is needed at runtime.

### Why not cuDNN?

GROMACS uses CUDA compute kernels for non-bonded interactions and PME. It does **not** use cuDNN (a deep-learning library). Dropping cuDNN from the base image avoids ~2 GB of unnecessary layers.

---

## Available tags

Tags follow the pattern `<gromacs-version>-cuda<cuda-version>`:

| Tag | GROMACS | CUDA |
|-----|---------|------|
| `2025.2-cuda12.8.1` | 2025.2 | 12.8.1 |

---

## Pull & run

```bash
# Pull
docker pull ghcr.io/jinzhanglab/gmxpytools/gromacs:2025.2-cuda12.8.1

# Interactive shell with GPU
docker run --gpus all -it \
  ghcr.io/jinzhanglab/gmxpytools/gromacs:2025.2-cuda12.8.1

# Run a GROMACS command on local files
docker run --gpus all --rm \
  -v "$(pwd)":/workspace \
  ghcr.io/jinzhanglab/gmxpytools/gromacs:2025.2-cuda12.8.1 \
  gmx mdrun -v -deffnm md

# Double-precision (CPU) — no --gpus needed
docker run --rm \
  -v "$(pwd)":/workspace \
  ghcr.io/jinzhanglab/gmxpytools/gromacs:2025.2-cuda12.8.1 \
  gmx_d mdrun -v -deffnm md
```

### Prerequisites on the host machine

| Requirement | Notes |
|-------------|-------|
| Docker Engine ≥ 20.10 | <https://docs.docker.com/engine/install/> |
| nvidia-container-toolkit | <https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html> |
| NVIDIA driver ≥ 525 (for CUDA 12.x) | `nvidia-smi` should report the driver version |

---

## Build locally

```bash
# Default (GROMACS 2025.2, CUDA 12.8.1)
docker build -t gromacs:local docker/gromacs/

# Custom versions
docker build \
  --build-arg GROMACS_VERSION=2024.4 \
  --build-arg CUDA_VERSION=12.6.3 \
  -t gromacs:2024.4-cuda12.6.3 \
  docker/gromacs/
```

### Build arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `GROMACS_VERSION` | `2025.2` | GROMACS version tag as it appears on the FTP server |
| `CUDA_VERSION` | `12.8.1` | CUDA version (must match an `nvidia/cuda` image tag) |
| `UBUNTU_VERSION` | `ubuntu22.04` | Ubuntu release used as base |

---

## CI/CD

Workflow: `.github/workflows/docker-gromacs.yml`

| Trigger | Action |
|---------|--------|
| Push to `main`/`master` with changes in `docker/gromacs/` | Build + push to GHCR |
| GitHub Release published | Build + push to GHCR |
| `workflow_dispatch` | Build + push with custom GROMACS/CUDA versions |

---

## Verify the image

After pulling, confirm both executables are present and working:

```bash
docker run --rm ghcr.io/jinzhanglab/gmxpytools/gromacs:2025.2-cuda12.8.1 \
  bash -c "gmx --version && echo '---' && gmx_d --version"
```

For GPU verification (requires host GPU + nvidia-container-toolkit):

```bash
docker run --gpus all --rm ghcr.io/jinzhanglab/gmxpytools/gromacs:2025.2-cuda12.8.1 \
  gmx mdrun -gpu_id 0 -h
```

# gmx_MMPBSA Docker Image

**Location**: `docker/gmx-mmpbsa/Dockerfile`  
**Published to**: `ghcr.io/jinzhanglab/gmxpytools/gmx-mmpbsa`

---

## Overview

A multi-stage Docker image that pairs a source-compiled [GROMACS](https://www.gromacs.org/) with [gmx_MMPBSA](https://valdes-tresanco-ms.github.io/gmx_MMPBSA/) and [AmberTools](https://ambermd.org/AmberTools.php) to perform end-state free energy calculations (MM-PB/GBSA).

### Why a separate image?

The latest GROMACS releases (2025.x) are not yet fully supported by gmx_MMPBSA 1.6.x and some of its AmberTools dependencies. Instead of downgrading the simulation image, **a dedicated image pins a GROMACS version that is verified compatible** with gmx_MMPBSA.

Recommended two-image workflow:

| Stage | Image | Purpose |
|-------|-------|---------|
| MD simulation | `gmxpytools/gromacs` (latest) | GPU-accelerated production runs |
| MMPBSA analysis | `gmxpytools/gmx-mmpbsa` (this image) | Trajectory post-processing, free energy |

Data exchange happens through a host-mounted volume (`-v`).

### Contents

| Tool | Source | Purpose |
|------|--------|---------|
| `gmx` | Compiled from source (GROMACS_VERSION) | Trajectory conversion, energy re-calculation |
| `gmx_MMPBSA` | conda-forge | MM-PB(GB)SA free energy calculation |
| `ante-MMPBSA.py` | AmberTools (conda-forge dep) | Topology preparation for MMPBSA |
| `MMPBSA.py` | AmberTools (conda-forge dep) | Legacy AmberTools MMPBSA interface |

---

## Image design

### Multi-stage build

```
Stage 1 (builder): ubuntu:22.04
  └─ Installs build tools, compilers, CMake (Kitware PPA), OpenMPI dev
  └─ Downloads + compiles gmx (single-precision, CPU-only, FFTW built-in)

Stage 2 (runtime): ubuntu:22.04  ← same glibc as builder
  └─ Installs Miniconda → conda-forge: gmx_mmpbsa + AmberTools
  └─ Copies /usr/local/gromacs from Stage 1
  └─ /usr/local/gromacs/bin precedes /opt/conda/bin in PATH
     → our compiled gmx is used; conda's gmx (dep of gmx_mmpbsa) is shadowed
```

### Why CPU-only GROMACS?

gmx_MMPBSA uses GROMACS only for trajectory format conversion and energy recomputation — short, single-frame operations that are CPU-bound. There is no benefit to including CUDA in this image, which keeps the image size significantly smaller.

### FFTW

Built by GROMACS itself (`GMX_BUILD_OWN_FFTW=ON`) and statically linked — no `libfftw3` package is needed at runtime.

---

## Available tags

Tags follow the pattern `<gromacs-version>-mmpbsa<mmpbsa-version>`:

| Tag | GROMACS | gmx_MMPBSA |
|-----|---------|-----------|
| `2024.4-mmpbsa1.6.3` | 2024.4 | 1.6.3 |

---

## Pull & run

```bash
# Pull (no login required — image is public)
docker pull ghcr.io/jinzhanglab/gmxpytools/gmx-mmpbsa:2024.4-mmpbsa1.6.3

# Verify tools are available
docker run --rm ghcr.io/jinzhanglab/gmxpytools/gmx-mmpbsa:2024.4-mmpbsa1.6.3 \
  bash -c "gmx --version && gmx_MMPBSA --version"

# Interactive shell (mount your working directory)
docker run --rm -it \
  -v "$(pwd)":/workspace \
  ghcr.io/jinzhanglab/gmxpytools/gmx-mmpbsa:2024.4-mmpbsa1.6.3

# Run gmx_MMPBSA on local files (all input/output in $(pwd))
docker run --rm \
  -v "$(pwd)":/workspace \
  ghcr.io/jinzhanglab/gmxpytools/gmx-mmpbsa:2024.4-mmpbsa1.6.3 \
  gmx_MMPBSA -O -i mmgbsa.in \
             -cs md.tpr \
             -ct md.xtc \
             -cp topol.top \
             -ci index.ndx \
             -co complex.prmtop \
             -o FINAL_RESULTS_MMPBSA.dat \
             -eo FINAL_RESULTS_MMPBSA.csv
```

!!! warning "Troubleshooting `docker pull` denied"
    GHCR packages are private by default. The CI workflow attempts to set the package
    public automatically after each push.

    If you still see `denied: denied`, the org admin must set visibility manually:

    1. Go to the package settings:  
       <https://github.com/orgs/JinZhangLab/packages/container/gmxpytools%2Fgmx-mmpbsa/settings>
    2. Scroll to **Danger Zone** → **Change visibility** → select **Public** → confirm.

    Alternatively, authenticate first:
    ```bash
    echo "<YOUR_GITHUB_PAT>" | docker login ghcr.io -u <your-github-username> --password-stdin
    docker pull ghcr.io/jinzhanglab/gmxpytools/gmx-mmpbsa:2024.4-mmpbsa1.6.3
    ```

### Prerequisites on the host machine

| Requirement | Notes |
|-------------|-------|
| Docker Engine ≥ 20.10 | <https://docs.docker.com/engine/install/> |
| No GPU required | All gmx_MMPBSA operations are CPU-only |

---

## Typical two-image workflow

```bash
# ── Step 1: Run MD simulation with the latest GPU-accelerated GROMACS ──────
docker run --gpus all --rm \
  -v "$(pwd)":/workspace \
  ghcr.io/jinzhanglab/gmxpytools/gromacs:2025.2-cuda12.8.1 \
  gmx mdrun -v -deffnm md

# ── Step 2: MM-PB(GB)SA analysis with the compatible image ─────────────────
docker run --rm \
  -v "$(pwd)":/workspace \
  ghcr.io/jinzhanglab/gmxpytools/gmx-mmpbsa:2024.4-mmpbsa1.6.3 \
  gmx_MMPBSA -O -i mmgbsa.in \
             -cs md.tpr \
             -ct md.xtc \
             -cp topol.top \
             -ci index.ndx \
             -co complex.prmtop \
             -o FINAL_RESULTS_MMPBSA.dat \
             -eo FINAL_RESULTS_MMPBSA.csv
```

Trajectory files (`md.tpr`, `md.xtc`, `topol.top`, `index.ndx`) produced in Step 1 are consumed in Step 2 through the shared host directory.

---

## Build locally

```bash
# Default (GROMACS 2024.4, gmx_MMPBSA 1.6.3)
docker build -t gmx-mmpbsa:local docker/gmx-mmpbsa/

# Custom versions
docker build \
  --build-arg GROMACS_VERSION=2023.5 \
  --build-arg GMX_MMPBSA_VERSION=1.6.3 \
  -t gmx-mmpbsa:2023.5-mmpbsa1.6.3 \
  docker/gmx-mmpbsa/
```

### Build arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `GROMACS_VERSION` | `2024.4` | GROMACS version tag as it appears on the FTP server |
| `GMX_MMPBSA_VERSION` | `1.6.3` | gmx_MMPBSA version available on conda-forge |

---

## CI/CD

Workflow: `.github/workflows/docker-gmx-mmpbsa.yml`

| Trigger | Action |
|---------|--------|
| Push to `main`/`master` with changes in `docker/gmx-mmpbsa/` | Build + push to GHCR |
| GitHub Release published | Build + push to GHCR |
| `workflow_dispatch` | Build + push with custom GROMACS/gmx_MMPBSA versions |

---

## Verify the image

After pulling, confirm both tools are present and working:

```bash
docker run --rm ghcr.io/jinzhanglab/gmxpytools/gmx-mmpbsa:2024.4-mmpbsa1.6.3 \
  bash -c "gmx --version && gmx_MMPBSA --version"
```

Expected output will show GROMACS version information followed by gmx_MMPBSA version details.

---

## gmx_MMPBSA compatibility reference

| gmx_MMPBSA | Supported GROMACS versions |
|-----------|---------------------------|
| 1.6.x | 2020.x – 2024.x |
| 1.5.x | 2018.x – 2022.x |

See the [official compatibility table](https://valdes-tresanco-ms.github.io/gmx_MMPBSA/dev/compatibility/) for the full matrix.

# gmxPyTools

**Version: 0.1.0-alpha.1**

A growing toolkit of Docker images, Python scripts, and automation workflows that streamline [GROMACS](https://www.gromacs.org/) molecular dynamics simulations.

---

## What's inside

| Component | Description |
|-----------|-------------|
| [`docker/gromacs`](docker/gromacs.md) | NVIDIA CUDA–accelerated GROMACS image with `gmx` (GPU) and `gmx_d` (CPU) |
| [`scripts/convertPar2GmxTop.py`](scripts/convert-charmm-to-gromacs.md) | Convert MATCH/CHARMM parameter files to GROMACS topology |

---

## Quick start

```bash
# Pull the pre-built image from GitHub Container Registry
docker pull ghcr.io/jinzhanglab/gmxpytools/gromacs:2025.2-cuda12.8.1

# Open an interactive shell (GPU required)
docker run --gpus all -it ghcr.io/jinzhanglab/gmxpytools/gromacs:2025.2-cuda12.8.1

# Or run a one-off GROMACS command
docker run --gpus all --rm \
  -v "$(pwd)":/workspace \
  ghcr.io/jinzhanglab/gmxpytools/gromacs:2025.2-cuda12.8.1 \
  gmx mdrun -v -deffnm md
```

---

## Project links

- **Source**: <https://github.com/JinZhangLab/gmxPyTools>
- **Packages**: <https://ghcr.io/jinzhanglab/gmxpytools>
- **Architecture guide**: [ARCHITECTURE.md](ARCHITECTURE.md)

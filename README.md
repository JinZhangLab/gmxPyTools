# gmxPyTools

**Version: 0.1.0-alpha.1**

A collection of tools, scripts, and Docker images to streamline [GROMACS](https://www.gromacs.org/) molecular dynamics simulations.

---

## Project Structure

```
gmxPyTools/
├── docker/                   # Docker images
│   └── gromacs/              # GROMACS image with NVIDIA CUDA support
│       └── Dockerfile
├── scripts/                  # Python utility scripts
│   └── convertPar2GmxTop.py  # Convert CHARMM (MATCH) files to GROMACS topology
└── .github/
    └── workflows/
        └── docker-gromacs.yml  # CI/CD: build & push GROMACS image to GHCR
```

New features can be added as additional scripts in `scripts/` or additional Docker images under `docker/<name>/`, without affecting existing functionality.

---

## Docker Images

### GROMACS with CUDA (`docker/gromacs`)

A GROMACS image built on `nvidia/cuda` that provides **both**:

| Binary  | Precision | GPU support |
|---------|-----------|-------------|
| `gmx`   | Single    | ✅ CUDA     |
| `gmx_d` | Double    | ❌ (CUDA does not support double-precision MD) |

#### Pull from GHCR

Images are versioned by GROMACS version and CUDA version:

```bash
# GROMACS 2025.2 with CUDA 12.8.1
docker pull ghcr.io/jinzhanglab/gmxpytools/gromacs:2025.2-cuda12.8.1
```

#### Build locally

```bash
# Default versions
docker build -t gromacs:local docker/gromacs/

# Custom GROMACS and CUDA versions
docker build \
  --build-arg GROMACS_VERSION=2024.4 \
  --build-arg CUDA_VERSION=12.6.3 \
  -t gromacs:2024.4-cuda12.6.3 \
  docker/gromacs/
```

#### Run

```bash
# Interactive shell with GPU access
docker run --gpus all -it ghcr.io/jinzhanglab/gmxpytools/gromacs:2025.2-cuda12.8.1

# Run a GROMACS command directly
docker run --gpus all --rm \
  -v $(pwd):/workspace \
  ghcr.io/jinzhanglab/gmxpytools/gromacs:2025.2-cuda12.8.1 \
  gmx grompp -f md.mdp -c conf.gro -p topol.top -o topol.tpr
```

---

## Python Scripts

### `scripts/convertPar2GmxTop.py`

Converts MATCH-generated CHARMM parameter files (RTF, PRM, PAR) and a PDB structure to GROMACS-compatible topology (`.top`) and coordinate (`.gro`) files using [ParmEd](https://parmed.github.io/ParmEd/).

**Requirements:**
```bash
pip install parmed
```

**Usage:**
```bash
python scripts/convertPar2GmxTop.py \
  --pdb molecule.pdb \
  --rtf molecule.rtf \
  --prm molecule.prm \
  --par molecule.par \
  --output ./output
```

---

## CI/CD

The GitHub Actions workflow `.github/workflows/docker-gromacs.yml` automatically builds and pushes the GROMACS Docker image to [GitHub Container Registry (GHCR)](https://ghcr.io) on:

- Push to `main`/`master` when `docker/gromacs/` changes
- Git tags matching `v*`
- Manual trigger (`workflow_dispatch`) — allows selecting GROMACS and CUDA versions

---

## Versioning

This project follows [Semantic Versioning](https://semver.org/):

- Pre-release: `0.x.y-alpha.z` / `0.x.y-beta.z`
- Stable release: `1.0.0` and above

Current version: **0.1.0-alpha.1**

---

## License

LGPL-2.1 (following GROMACS)

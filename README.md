# gmxPyTools

**Version: 0.1.0-alpha.1** · [Documentation](https://jinzhanglab.github.io/gmxPyTools/) · [Packages (GHCR)](https://ghcr.io/jinzhanglab/gmxpytools)

A growing toolkit of Docker images, Python scripts, and automation workflows that streamline [GROMACS](https://www.gromacs.org/) molecular dynamics simulations.

---

## Project Structure

```
gmxPyTools/
├── docker/                        # Docker images (one sub-directory per image)
│   ├── gromacs/
│   │   └── Dockerfile             # Multi-stage: devel builder → runtime final
│   └── gmx-mmpbsa/
│       └── Dockerfile             # Multi-stage: CPU GROMACS builder → conda runtime
├── scripts/                       # Standalone Python utilities
│   └── convertPar2GmxTop.py       # Convert CHARMM (MATCH) files to GROMACS topology
├── docs/                          # MkDocs documentation source
│   ├── index.md
│   ├── ARCHITECTURE.md            # Binding architectural constraints
│   ├── docker/
│   │   ├── gromacs.md
│   │   └── gmx-mmpbsa.md
│   └── scripts/convert-charmm-to-gromacs.md
├── mkdocs.yml                     # MkDocs configuration
└── .github/
    └── workflows/
        ├── docker-gromacs.yml     # CI/CD: build & push GROMACS image to GHCR
        ├── docker-gmx-mmpbsa.yml  # CI/CD: build & push gmx_MMPBSA image to GHCR
        └── docs.yml               # CI/CD: deploy docs to GitHub Pages
```

New features are added as self-contained units — new Docker images under `docker/<name>/` or new scripts in `scripts/` — without modifying existing files. See [ARCHITECTURE.md](docs/ARCHITECTURE.md) for the full set of rules.

---

## Docker Images

### GROMACS with CUDA (`docker/gromacs`)

A **multi-stage** GROMACS image (compile in `devel`, ship in lean `runtime`) providing:

| Binary  | Precision | GPU support |
|---------|-----------|-------------|
| `gmx`   | Single    | ✅ CUDA     |
| `gmx_d` | Double    | ❌ CPU only (CUDA does not support double-precision MD) |

> **Image size note**: By using a `runtime` base (instead of `devel`) and omitting cuDNN (which GROMACS does not use), the final image is substantially smaller than a naive single-stage build.

#### Host requirements

| Requirement | Notes |
|-------------|-------|
| Docker Engine ≥ 20.10 | <https://docs.docker.com/engine/install/> |
| nvidia-container-toolkit | <https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html> |
| NVIDIA driver ≥ 525 (for CUDA 12.x) | Verify with `nvidia-smi` |

#### Quick start

```bash
# Pull (no login required — image is public)
docker pull ghcr.io/jinzhanglab/gmxpytools/gromacs:2025.2-cuda12.8.1

# Verify both executables work (no GPU needed for this check)
docker run --rm ghcr.io/jinzhanglab/gmxpytools/gromacs:2025.2-cuda12.8.1 \
  bash -c "gmx --version && echo '---' && gmx_d --version"

# Interactive shell with GPU
docker run --gpus all -it ghcr.io/jinzhanglab/gmxpytools/gromacs:2025.2-cuda12.8.1

# Run a GROMACS command on local files
docker run --gpus all --rm \
  -v "$(pwd)":/workspace \
  ghcr.io/jinzhanglab/gmxpytools/gromacs:2025.2-cuda12.8.1 \
  gmx mdrun -v -deffnm md
```

> **Troubleshooting `docker pull` denied**  
> GHCR packages are private by default. The CI workflow attempts to set the package public automatically.  
> If you still see "denied", the org admin needs to manually set visibility:  
> 1. Go to <https://github.com/orgs/JinZhangLab/packages/container/gmxpytools%2Fgromacs/settings>  
> 2. Scroll to **Danger Zone → Change visibility → Public**  
>  
> Alternatively, log in first: `docker login ghcr.io -u <your-github-username>`

#### Build locally with custom versions

```bash
docker build \
  --build-arg GROMACS_VERSION=2024.4 \
  --build-arg CUDA_VERSION=12.6.3 \
  -t gromacs:2024.4-cuda12.6.3 \
  docker/gromacs/
```

See [docs/docker/gromacs.md](docs/docker/gromacs.md) for full documentation.

---

### gmx_MMPBSA with GROMACS (`docker/gmx-mmpbsa`)

A dedicated image for **MM-PB(GB)SA free energy calculations** combining a source-compiled GROMACS (verified compatible with gmx_MMPBSA) with [gmx_MMPBSA](https://valdes-tresanco-ms.github.io/gmx_MMPBSA/) and [AmberTools](https://ambermd.org/AmberTools.php) installed via conda-forge.

> **Why a separate image?**  
> The latest GROMACS (2025.x) is not yet fully supported by gmx_MMPBSA 1.6.x.  
> Use the `gromacs` image for GPU-accelerated MD simulations and this image for post-analysis — sharing data through a host-mounted volume.

| Tool | Version | Purpose |
|------|---------|---------|
| `gmx` | 2024.4 (default) | Trajectory processing (CPU-only) |
| `gmx_MMPBSA` | 1.6.3 (default) | MM-PB(GB)SA free energy |
| AmberTools | via conda-forge | Topology prep (`ante-MMPBSA.py`) |

#### Host requirements

| Requirement | Notes |
|-------------|-------|
| Docker Engine ≥ 20.10 | <https://docs.docker.com/engine/install/> |
| No GPU required | gmx_MMPBSA analysis is CPU-only |

#### Quick start

```bash
# Pull (no login required — image is public)
docker pull ghcr.io/jinzhanglab/gmxpytools/gmx-mmpbsa:2024.4-mmpbsa1.6.3

# Verify tools work
docker run --rm ghcr.io/jinzhanglab/gmxpytools/gmx-mmpbsa:2024.4-mmpbsa1.6.3 \
  bash -c "gmx --version && gmx_MMPBSA --version"

# Interactive shell with local files mounted
docker run --rm -it \
  -v "$(pwd)":/workspace \
  ghcr.io/jinzhanglab/gmxpytools/gmx-mmpbsa:2024.4-mmpbsa1.6.3

# Run MM-GBSA analysis on local trajectory files
docker run --rm \
  -v "$(pwd)":/workspace \
  ghcr.io/jinzhanglab/gmxpytools/gmx-mmpbsa:2024.4-mmpbsa1.6.3 \
  gmx_MMPBSA -O -i mmgbsa.in -cs md.tpr -ct md.xtc \
             -cp topol.top -ci index.ndx \
             -co complex.prmtop \
             -o FINAL_RESULTS_MMPBSA.dat \
             -eo FINAL_RESULTS_MMPBSA.csv
```

#### Typical two-image workflow

```bash
# Step 1 — GPU-accelerated MD with the latest GROMACS
docker run --gpus all --rm \
  -v "$(pwd)":/workspace \
  ghcr.io/jinzhanglab/gmxpytools/gromacs:2025.2-cuda12.8.1 \
  gmx mdrun -v -deffnm md

# Step 2 — MMPBSA analysis (no GPU needed)
docker run --rm \
  -v "$(pwd)":/workspace \
  ghcr.io/jinzhanglab/gmxpytools/gmx-mmpbsa:2024.4-mmpbsa1.6.3 \
  gmx_MMPBSA -O -i mmgbsa.in -cs md.tpr -ct md.xtc \
             -cp topol.top -ci index.ndx \
             -co complex.prmtop \
             -o FINAL_RESULTS_MMPBSA.dat \
             -eo FINAL_RESULTS_MMPBSA.csv
```

#### Build locally with custom versions

```bash
docker build \
  --build-arg GROMACS_VERSION=2023.5 \
  --build-arg GMX_MMPBSA_VERSION=1.6.3 \
  -t gmx-mmpbsa:2023.5-mmpbsa1.6.3 \
  docker/gmx-mmpbsa/
```

See [docs/docker/gmx-mmpbsa.md](docs/docker/gmx-mmpbsa.md) for full documentation.

---

## Python Scripts

### `scripts/convertPar2GmxTop.py`

Converts MATCH-generated CHARMM parameter files (RTF, PRM, PAR) and a PDB structure to GROMACS-compatible topology (`.top`) and coordinate (`.gro`) files using [ParmEd](https://parmed.github.io/ParmEd/).

```bash
pip install parmed

python scripts/convertPar2GmxTop.py \
  --pdb molecule.pdb --rtf molecule.rtf \
  --prm molecule.prm --par molecule.par \
  --output ./output
```

See [docs/scripts/convert-charmm-to-gromacs.md](docs/scripts/convert-charmm-to-gromacs.md) for full documentation.

---

## CI/CD

| Workflow | Trigger | Action |
|----------|---------|--------|
| `docker-gromacs.yml` | Push to main (docker/gromacs/ changes), Release published, Manual | Build & push GROMACS image to GHCR |
| `docker-gmx-mmpbsa.yml` | Push to main (docker/gmx-mmpbsa/ changes), Release published, Manual | Build & push gmx_MMPBSA image to GHCR |
| `docs.yml` | Push to main (docs/ changes), Manual | Build & deploy docs to GitHub Pages |

---

## Versioning

Follows [Semantic Versioning](https://semver.org/): `0.x.y-alpha.z` (pre-release) → `1.0.0` (stable).

Current: **0.1.0-alpha.1**

---

## License

LGPL-2.1 (following GROMACS)

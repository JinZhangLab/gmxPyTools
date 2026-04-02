# gmxPyTools

**Version: 0.1.0-alpha.1** · [Documentation](https://jinzhanglab.github.io/gmxPyTools/) · [Packages (GHCR)](https://ghcr.io/jinzhanglab/gmxpytools)

A growing toolkit of Docker images, Python scripts, and automation workflows that streamline [GROMACS](https://www.gromacs.org/) molecular dynamics simulations.

---

## Project Structure

```
gmxPyTools/
├── docker/                        # Docker images (one sub-directory per image)
│   └── gromacs/
│       └── Dockerfile             # Multi-stage: devel builder → runtime final
├── scripts/                       # Standalone Python utilities
│   └── convertPar2GmxTop.py       # Convert CHARMM (MATCH) files to GROMACS topology
├── docs/                          # MkDocs documentation source
│   ├── index.md
│   ├── ARCHITECTURE.md            # Binding architectural constraints
│   ├── docker/gromacs.md
│   └── scripts/convert-charmm-to-gromacs.md
├── mkdocs.yml                     # MkDocs configuration
└── .github/
    └── workflows/
        ├── docker-gromacs.yml     # CI/CD: build & push GROMACS image to GHCR
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
| `docs.yml` | Push to main (docs/ changes), Manual | Build & deploy docs to GitHub Pages |

---

## Versioning

Follows [Semantic Versioning](https://semver.org/): `0.x.y-alpha.z` (pre-release) → `1.0.0` (stable).

Current: **0.1.0-alpha.1**

---

## License

LGPL-2.1 (following GROMACS)

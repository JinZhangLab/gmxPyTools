# convertPar2GmxTop — CHARMM → GROMACS topology converter

**Location**: `scripts/convertPar2GmxTop.py`

---

## Overview

Converts files produced by [MATCH](http://brooks.chem.lsa.umich.edu/index.php?matchserver=showcase&page_id=198&tab_id=195) (Multipurpose Atom-Typer for CHARMM) into GROMACS-compatible files:

| Input | Output |
|-------|--------|
| `<mol>.pdb` (structure from MATCH) | `<mol>.gro` (GROMACS coordinate file) |
| `<mol>.rtf` (residue topology) | `<mol>.top` (GROMACS topology) |
| `<mol>.prm` + `<mol>.par` (parameters) | |

Uses [ParmEd](https://parmed.github.io/ParmEd/) for topology translation.

---

## Requirements

```bash
pip install parmed
```

---

## Usage

```bash
python scripts/convertPar2GmxTop.py \
  --pdb  cholesterol.pdb \
  --rtf  cholesterol.rtf \
  --prm  cholesterol.prm \
  --par  cholesterol.par \
  --output ./output/
```

Output files will be named after the PDB file stem (e.g. `cholesterol.top`, `cholesterol.gro`).

### Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `--pdb`  | ✅ | PDB structure file (MATCH output) |
| `--rtf`  | ✅ | CHARMM RTF residue topology file (MATCH output) |
| `--prm`  | ✅ | CHARMM PRM parameter file (MATCH output) |
| `--par`  | ✅ | CHARMM PAR parameter file (MATCH output) |
| `--output` | ❌ | Output directory (default: current directory) |

---

## What the script does

1. Loads the MATCH-generated CHARMM parameter set (RTF + PRM + PAR).
2. Detects the residue name automatically from the parameter set.
3. Assigns atom types from the template residue to every atom in the structure.
4. Reconstructs all missing bonds from the template, then derives angles, dihedrals, and impropers.
5. Builds a CHARMM PSF topology object and loads force-field parameters into it.
6. Writes GROMACS `.top` and `.gro` files using ParmEd.

---

## Typical workflow

```
MATCH server → molecule.rtf / .prm / .par / .pdb
                    ↓
          convertPar2GmxTop.py
                    ↓
         molecule.top + molecule.gro
                    ↓
          GROMACS simulation
```

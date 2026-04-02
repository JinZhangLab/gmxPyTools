# gmxPyTools — Architecture & Contribution Constraints

**Version applicable from**: 0.1.0-alpha.1  
**Status**: Binding — all contributors and automated tooling must comply.

---

## 1. Purpose and philosophy

gmxPyTools is a **modular, plug-in–style toolkit** that makes GROMACS molecular dynamics simulations easier. The guiding principle is:

> **Every feature is self-contained. Adding a feature must never require modifying existing features.**

Concretely this means:
- New Docker images, scripts, or workflows are **additions**, not mutations of existing files.
- Shared logic (if any ever arises) lives in a dedicated `lib/` directory — never inside a feature file.
- CI/CD pipelines are **per-feature** (one workflow per Docker image, one per major script category).

---

## 2. Mandated directory layout

```
gmxPyTools/
├── docker/                  # One sub-directory per Docker image
│   └── <image-name>/
│       └── Dockerfile       # Required; no other build file name is accepted
│
├── scripts/                 # Standalone Python scripts (no shared state between scripts)
│   └── <PascalCase>.py
│
├── docs/                    # Documentation (MkDocs source)
│   ├── index.md
│   ├── ARCHITECTURE.md      # This file (kept in docs/ so it renders in the site)
│   ├── docker/
│   │   └── <image-name>.md  # One page per Docker image
│   └── scripts/
│       └── <kebab-case>.md  # One page per script
│
├── mkdocs.yml               # MkDocs configuration (do not rename)
│
└── .github/
    └── workflows/
        ├── docker-<image-name>.yml   # One workflow per Docker image
        └── docs.yml                  # Docs deployment workflow (do not rename)
```

### Rules

1. **No new top-level directories** without updating this document and `mkdocs.yml`.
2. **No shared Python modules** between scripts unless placed in `scripts/lib/` with its own documentation page.
3. **Dockerfile naming**: always `Dockerfile` (capital D, no extension). No `docker-compose.yml` unless a feature explicitly requires multi-container orchestration and it is documented.

---

## 3. Adding a new Docker image

Checklist — every item is required before merging:

- [ ] Create `docker/<image-name>/Dockerfile` using the **multi-stage pattern** (builder stage → runtime stage).
  - Builder stage: use the largest required base (e.g. `*-devel-*`).
  - Runtime stage: use the smallest sufficient base (e.g. `*-runtime-*` or `ubuntu`).
  - Do **not** include build-time tools (compilers, headers) in the runtime stage.
  - Do **not** run `make check` / test suites inside the Dockerfile — tests require runtime resources (GPU, network) unavailable during CI builds.
- [ ] All user-selectable parameters (version numbers, etc.) exposed as `ARG` with sensible defaults.
- [ ] Required `LABEL` metadata (see §6).
- [ ] Create `.github/workflows/docker-<image-name>.yml` following the trigger pattern in §7.
- [ ] Create `docs/docker/<image-name>.md` covering: purpose, image design, available tags, run examples, build args, CI/CD notes, and verification steps.
- [ ] Update `mkdocs.yml` nav to include the new docs page.
- [ ] Update `README.md` Docker Images table.

---

## 4. Adding a new Python script

Checklist:

- [ ] Place the file in `scripts/` with a descriptive `PascalCase` name.
- [ ] The script must be runnable standalone (`if __name__ == "__main__": ...` with `argparse`).
- [ ] No hard-coded paths or output filenames — derive them from input arguments.
- [ ] Include a module-level docstring describing purpose, inputs, outputs, and usage example.
- [ ] Create `docs/scripts/<kebab-case>.md` (see existing pages for the required structure).
- [ ] Update `mkdocs.yml` nav.
- [ ] Update `README.md` Python Scripts table.

---

## 5. Documentation rules

- **Docs live alongside code.** If you change a Dockerfile or script, you must update the corresponding `docs/` page in the same PR.
- **`mkdocs.yml` nav is the single source of truth** for what is documented. Every file in `docs/` must be listed there.
- Docs are built and deployed automatically to GitHub Pages by `.github/workflows/docs.yml` on every push to `main`/`master`.
- Docs must be written in Markdown. No raw HTML except in MkDocs admonitions.
- Code blocks must specify the language for syntax highlighting.

---

## 6. Required Docker image labels

Every `Dockerfile` runtime stage must include:

```dockerfile
LABEL org.opencontainers.image.title="<human-readable name>"
LABEL org.opencontainers.image.description="<one-sentence description>"
LABEL org.opencontainers.image.source="https://github.com/JinZhangLab/gmxPyTools"
LABEL org.opencontainers.image.licenses="<SPDX identifier>"
LABEL <feature>.version="${<FEATURE>_VERSION}"   # e.g. gromacs.version
LABEL cuda.version="${CUDA_VERSION}"              # if CUDA is involved
```

---

## 7. CI/CD workflow conventions

| Workflow file | Trigger | Purpose |
|---------------|---------|---------|
| `docker-<name>.yml` | `push` (paths filter) + `release` (published) + `workflow_dispatch` | Build & push image to GHCR |
| `docs.yml` | `push` (paths: docs/**, mkdocs.yml) + `workflow_dispatch` | Build & deploy docs to GitHub Pages |

Rules:
- Use `docker/login-action`, `docker/setup-buildx-action`, `docker/metadata-action`, `docker/build-push-action` — the standard Docker GitHub Actions suite.
- Always specify `permissions` explicitly (`contents: read`, `packages: write`).
- Always use `cache-from: type=gha` / `cache-to: type=gha,mode=max` for layer caching.
- Pin action versions with `@vN` (major version pin), not `@sha`.
- Do **not** combine `push.branches`+`paths` with `push.tags` in the same event block — use the `release` event for release-triggered builds.

---

## 8. Versioning

This project follows [Semantic Versioning 2.0.0](https://semver.org/).

| Phase | Version pattern | Example |
|-------|----------------|---------|
| Pre-alpha development | `0.x.y-alpha.z` | `0.1.0-alpha.1` |
| Beta (feature-complete, testing) | `0.x.y-beta.z` | `0.2.0-beta.1` |
| First stable release | `1.0.0` | — |
| Stable patches | `1.x.y` | `1.0.1` |

- Version is recorded in `README.md` (top of file).
- Docker image tags encode the **tool version** (e.g. GROMACS version) and the **CUDA version**, not the gmxPyTools version. This allows independent versioning of images and project.
- Create a GitHub Release for every version bump. The release event triggers Docker image rebuilds.

---

## 9. Branch strategy

| Branch | Purpose |
|--------|---------|
| `main` | Stable, deployable state. Protected. |
| `feature/<short-description>` | New feature development |
| `fix/<short-description>` | Bug fixes |
| `docs/<short-description>` | Documentation-only changes |
| `copilot/<description>` | Automated agent branches |

- All merges to `main` via Pull Request.
- PRs must update docs if any user-facing behavior changes.

---

## 10. Extensibility checklist (summary)

When adding **any** new feature, verify:

1. ✅ Self-contained in its own directory/file — no edits to unrelated files (except `README.md`, `mkdocs.yml`).
2. ✅ Documentation page created and added to `mkdocs.yml` nav.
3. ✅ CI/CD workflow created (for Docker images) or existing workflow extended (for scripts, if a test suite exists).
4. ✅ `README.md` table updated.
5. ✅ This document reviewed — if the new feature type does not fit any existing category, add a new section here.

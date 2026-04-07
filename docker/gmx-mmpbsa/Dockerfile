# gmx_MMPBSA Docker image — GROMACS + AmberTools + gmx_MMPBSA
# Multi-stage build: compiles a GROMACS version compatible with gmx_MMPBSA in
# the builder stage, then installs AmberTools and gmx_MMPBSA via conda in the
# lean runtime stage.
#
# Design rationale
# ─────────────────
# The latest GROMACS releases (2025.x) are not yet fully supported by
# gmx_MMPBSA and some of its AmberTools dependencies. Rather than coupling
# the simulation image (docker/gromacs) to an older GROMACS, we maintain a
# dedicated gmx-mmpbsa image that pins a GROMACS version known to work
# (2024.4 by default). Users run production MD with the latest GROMACS image
# and run MM-PB(GB)SA post-analysis with this image — sharing data via a
# host-mounted volume.
#
# Build arguments:
#   GROMACS_VERSION    - GROMACS source version (default: 2024.4)
#   GMX_MMPBSA_VERSION - gmx_MMPBSA conda-forge package version (default: 1.6.3)
#
# The final image contains:
#   gmx           — GROMACS single-precision, CPU-only  (trajectory processing)
#   gmx_MMPBSA    — MM-PB(GB)SA free-energy tool
#   ante-MMPBSA.py — AmberTools utility (topology preparation)
#   MMPBSA.py      — AmberTools utility (legacy interface)
#
# Example builds:
#   docker build -t gmx-mmpbsa:2024.4 docker/gmx-mmpbsa/
#   docker build --build-arg GROMACS_VERSION=2023.5 -t gmx-mmpbsa:2023.5 docker/gmx-mmpbsa/

# ──────────────────────────────────────────────────────────────────────────────
# Stage 1 – builder
# Compile GROMACS (CPU-only, single-precision) inside ubuntu:22.04 so the
# resulting binary is linked against the same glibc as the runtime stage.
# ──────────────────────────────────────────────────────────────────────────────
FROM ubuntu:22.04 AS builder

ARG GROMACS_VERSION=2024.4

ENV DEBIAN_FRONTEND=noninteractive

# Install build-time dependencies + CMake from the Kitware PPA (Ubuntu 22.04
# ships CMake 3.22 which is too old for GROMACS 2024+; Kitware provides ≥ 3.28).
RUN apt-get update && \
    apt-get install -y \
        apt-transport-https \
        ca-certificates \
        gnupg \
        wget \
        make \
        gcc \
        g++ \
        libopenmpi-dev && \
    wget -O - https://apt.kitware.com/keys/kitware-archive-latest.asc \
        | gpg --dearmor -o /usr/share/keyrings/kitware-archive-keyring.gpg && \
    echo 'deb [signed-by=/usr/share/keyrings/kitware-archive-keyring.gpg] https://apt.kitware.com/ubuntu/ jammy main' \
        | tee /etc/apt/sources.list.d/kitware.list > /dev/null && \
    apt-get update && \
    apt-get install -y cmake && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /gromacs

# Download GROMACS source
RUN wget -q "http://ftp.gromacs.org/pub/gromacs/gromacs-${GROMACS_VERSION}.tar.gz" \
    && tar -xzf "gromacs-${GROMACS_VERSION}.tar.gz" \
    && rm "gromacs-${GROMACS_VERSION}.tar.gz"

# Build single-precision, CPU-only.
# gmx_MMPBSA analysis is entirely CPU-bound; no CUDA is required.
# GMX_BUILD_OWN_FFTW=ON statically links FFTW so no libfftw3 is needed at runtime.
RUN cmake -S "gromacs-${GROMACS_VERSION}" -B build_sp \
        -DGMX_BUILD_OWN_FFTW=ON \
        -DGMX_GPU=OFF \
        -DGMX_DOUBLE=OFF \
        -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_INSTALL_PREFIX=/usr/local/gromacs && \
    cmake --build build_sp -j "$(nproc)" && \
    cmake --install build_sp && \
    rm -rf build_sp "gromacs-${GROMACS_VERSION}"

# ──────────────────────────────────────────────────────────────────────────────
# Stage 2 – runtime
# Ubuntu 22.04 base (same glibc as builder) + Miniconda for AmberTools and
# gmx_MMPBSA, plus the compiled GROMACS binary copied from Stage 1.
# ──────────────────────────────────────────────────────────────────────────────
FROM ubuntu:22.04

ARG GROMACS_VERSION=2024.4
ARG GMX_MMPBSA_VERSION=1.6.3

LABEL org.opencontainers.image.title="gmx_MMPBSA with GROMACS"
LABEL org.opencontainers.image.description="gmx_MMPBSA end-state free energy calculator (MM-PB/GBSA) paired with a GROMACS installation that is verified compatible with gmx_MMPBSA 1.6.x."
LABEL org.opencontainers.image.source="https://github.com/JinZhangLab/gmxPyTools"
LABEL org.opencontainers.image.licenses="LGPL-2.1"
LABEL gromacs.version="${GROMACS_VERSION}"
LABEL gmx_mmpbsa.version="${GMX_MMPBSA_VERSION}"

ENV DEBIAN_FRONTEND=noninteractive
ENV OMPI_ALLOW_RUN_AS_ROOT=1
ENV OMPI_ALLOW_RUN_AS_ROOT_CONFIRM=1

# Install system runtime libraries required by GROMACS and OpenMPI.
# wget + ca-certificates are needed to bootstrap Miniconda.
RUN apt-get update && \
    apt-get install -y \
        wget \
        ca-certificates \
        libopenmpi3 \
        libgomp1 && \
    rm -rf /var/lib/apt/lists/*

# Install Miniconda into /opt/conda (batch, non-interactive).
RUN wget -q "https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh" \
        -O /tmp/miniconda.sh && \
    bash /tmp/miniconda.sh -b -p /opt/conda && \
    rm /tmp/miniconda.sh && \
    /opt/conda/bin/conda clean -afy

ENV PATH="/opt/conda/bin:${PATH}"

# Install AmberTools and gmx_MMPBSA from conda-forge.
# conda will also pull in its own GROMACS as a dependency of gmx_mmpbsa;
# that is intentional — our compiled GROMACS (copied below) overrides it via PATH.
RUN conda install -y -n base -c conda-forge \
        "gmx_mmpbsa=${GMX_MMPBSA_VERSION}" && \
    conda clean -afy

# Copy the compiled GROMACS installation from the builder stage.
# Placing /usr/local/gromacs/bin first in PATH ensures our version takes
# precedence over the GROMACS binary installed by conda.
COPY --from=builder /usr/local/gromacs /usr/local/gromacs

ENV PATH="/usr/local/gromacs/bin:/opt/conda/bin:${PATH}"

# Make GMXRC available in interactive shells
RUN echo 'source /usr/local/gromacs/bin/GMXRC' >> /etc/bash.bashrc

WORKDIR /workspace

CMD ["/bin/bash"]

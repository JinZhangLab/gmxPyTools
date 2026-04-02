# GROMACS Docker image with NVIDIA CUDA support
# Multi-stage build: compiles in a devel image, ships a lean runtime image.
#
# Build arguments:
#   GROMACS_VERSION  - GROMACS version to build (default: 2025.2)
#   CUDA_VERSION     - CUDA base image version (default: 12.8.1)
#   UBUNTU_VERSION   - Ubuntu base version (default: ubuntu22.04)
#
# The final image contains:
#   gmx    — single-precision, CUDA GPU-accelerated
#   gmx_d  — double-precision, CPU only (CUDA does not support double-precision MD)
#
# Example builds:
#   docker build -t gromacs:2025.2-cuda12.8.1 docker/gromacs/
#   docker build --build-arg GROMACS_VERSION=2024.4 -t gromacs:2024.4 docker/gromacs/

# ──────────────────────────────────────────────────────────────────────────────
# Global ARGs (available to all FROM instructions)
# ──────────────────────────────────────────────────────────────────────────────
ARG CUDA_VERSION=12.8.1
ARG UBUNTU_VERSION=ubuntu22.04

# ──────────────────────────────────────────────────────────────────────────────
# Stage 1 – builder
# Full CUDA devel image with compilers and headers needed to compile GROMACS.
# ──────────────────────────────────────────────────────────────────────────────
FROM nvidia/cuda:${CUDA_VERSION}-devel-${UBUNTU_VERSION} AS builder

# Re-declare ARG inside the stage so it is usable in RUN commands
ARG GROMACS_VERSION=2025.2

ENV DEBIAN_FRONTEND=noninteractive

# Install build-time dependencies + CMake from the Kitware PPA (Ubuntu 22.04 ships
# CMake 3.22 which is too old for GROMACS 2024+; Kitware provides ≥ 3.28).
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
    && rm  "gromacs-${GROMACS_VERSION}.tar.gz"

# Build single-precision (gmx) with CUDA GPU support.
# GMX_BUILD_OWN_FFTW=ON statically links FFTW so no libfftw3 is needed at runtime.
RUN cmake -S "gromacs-${GROMACS_VERSION}" -B build_single \
        -DGMX_BUILD_OWN_FFTW=ON \
        -DGMX_GPU=CUDA \
        -DGMX_DOUBLE=OFF \
        -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_INSTALL_PREFIX=/usr/local/gromacs && \
    cmake --build build_single -j "$(nproc)" && \
    cmake --install build_single && \
    rm -rf build_single

# Build double-precision (gmx_d) — CPU only.
RUN cmake -S "gromacs-${GROMACS_VERSION}" -B build_double \
        -DGMX_BUILD_OWN_FFTW=ON \
        -DGMX_GPU=OFF \
        -DGMX_DOUBLE=ON \
        -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_INSTALL_PREFIX=/usr/local/gromacs && \
    cmake --build build_double -j "$(nproc)" && \
    cmake --install build_double && \
    rm -rf build_double "gromacs-${GROMACS_VERSION}"

# ──────────────────────────────────────────────────────────────────────────────
# Stage 2 – runtime
# Lean CUDA runtime image; only the libraries needed to *run* GROMACS are kept.
# The nvidia/cuda:*-runtime image includes libcudart and cuFFT (math libraries).
# ──────────────────────────────────────────────────────────────────────────────
FROM nvidia/cuda:${CUDA_VERSION}-runtime-${UBUNTU_VERSION}

ARG GROMACS_VERSION=2025.2
ARG CUDA_VERSION=12.8.1

LABEL org.opencontainers.image.title="GROMACS with CUDA"
LABEL org.opencontainers.image.description="GROMACS molecular dynamics simulation software with NVIDIA CUDA GPU acceleration. Includes single-precision (gmx, GPU) and double-precision (gmx_d, CPU) builds."
LABEL org.opencontainers.image.source="https://github.com/JinZhangLab/gmxPyTools"
LABEL org.opencontainers.image.licenses="LGPL-2.1"
LABEL gromacs.version="${GROMACS_VERSION}"
LABEL cuda.version="${CUDA_VERSION}"

ENV DEBIAN_FRONTEND=noninteractive
ENV OMPI_ALLOW_RUN_AS_ROOT=1
ENV OMPI_ALLOW_RUN_AS_ROOT_CONFIRM=1

# Install only runtime libraries (no compilers, no build tools)
RUN apt-get update && \
    apt-get install -y \
        libopenmpi3 \
        libgomp1 \
        python3 && \
    rm -rf /var/lib/apt/lists/*

# Copy the compiled GROMACS installation from the builder stage
COPY --from=builder /usr/local/gromacs /usr/local/gromacs

ENV PATH="/usr/local/gromacs/bin:${PATH}"

# Make GMXRC available in interactive shells
RUN echo 'source /usr/local/gromacs/bin/GMXRC' >> /etc/bash.bashrc

WORKDIR /workspace

CMD ["/bin/bash"]
